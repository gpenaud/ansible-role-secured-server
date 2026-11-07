# gpenaud.secured-server

Ferme une machine Debian ou Ubuntu exposée sur Internet, et la garde fermée à chaque
passage. Un rôle, dix fichiers de tâches, chacun une chose :

| Fichier | Ce qu'il garantit |
|---|---|
| `packages.yml` | système à jour, quelques outils |
| `accounts.yml` | un compte de service avec la seule clé d'Ansible ; qui entre par SSH est dans `sshusers`, qui a sudo est dans `sudousers`, membres exacts |
| `ssh.yml` | clés seules, pas de root, pas de mot de passe, pas de tunnel, chiffrement moderne ; configuration validée par `sshd -t` avant d'être posée |
| `time.yml` | l'heure par `systemd-timesyncd`, aucun autre démon NTP |
| `updates.yml` | `unattended-upgrades` : sécurité et distribution, redémarrage pour un noyau à l'heure dite |
| `password-quality.yml` | `pam_pwquality` pour le mot de passe de sudo |
| `firewall.yml` | ufw : tout refusé en entrée sauf la liste (22 limité, 80, 443, 443/udp), tout permis en sortie |
| `crowdsec.yml` | CrowdSec et son bouncer iptables, collections `linux` et `sshd` |
| `auditing.yml` | AIDE quotidien, Lynis hebdomadaire ; rkhunter, ClamAV, logwatch sur demande |
| `mail.yml` | exim4 vers un relais authentifié, seulement si l'inventaire le donne |
| `cleanup.yml` | comptes par défaut des images retirés ; ce que les anciennes versions du rôle avaient posé est enlevé |

## Ce que l'inventaire donne

```yaml
secure_server_fqdn: infra03.alter-it.eu
secure_server_user_password: "{{ lookup('env', 'BECOME_PASS_INFRA03') }}"   # jamais vide sur une machine exposée
secure_server_ssh_key: "{{ lookup('file', '~/.ssh/infra03-ansible.pub') }}"
secure_server_ssh_allowed_users: ["operator", "gpenaud"]
secure_server_sudo_allowed_users: ["operator", "gpenaud"]
```

Les ports au-delà de 22, 80 et 443, les réseaux et routes (une machine Kubernetes) :
`secure_server_ufw_allowed_ports`, `secure_server_ufw_allowed_sources`,
`secure_server_ufw_allowed_routes`. Les ports que Docker publie ne passent pas par ici mais
par la chaîne `DOCKER-USER`, posée par le playbook qui installe Docker.

Le courrier sortant : `secure_server_mail_smarthost`, `secure_server_mail_user`,
`secure_server_mail_password`. Le mot de passe n'a pas de défaut et ne doit jamais être
écrit dans un dépôt. Sans relais, les comptes rendus restent dans la boîte de root.

Toutes les valeurs sont dans `defaults/main/`, commentées variable par variable.

## Ce qu'il faut savoir

- **Première passe** sur une image neuve : en root, `-e ansible_user=root`. Le rôle ferme root
  et ouvre le compte de service ; les passes suivantes se font avec lui.
- **Le compte connecté n'est jamais supprimé** : `cleanup.yml` retire `ubuntu` et `debian` sauf
  s'ils sont celui par lequel Ansible est entré.
- **AIDE** initialise sa base au premier passage : quelques minutes.
- **Lynis** écrit son score dans `/var/log/lynis.log` chaque dimanche : c'est la mesure à
  comparer d'un passage à l'autre, et avant d'ouvrir une machine au public.
- **CrowdSec** garde ses métriques sur `127.0.0.1:6060`, pour le collecteur de la machine.

## Vérifier de l'extérieur

```sh
nmap -p- <ip>                                  # trois ports : 22, 80, 443 (et 443/udp)
ssh -o PreferredAuthentications=password root@<ip>   # refusé
ssh -o PreferredAuthentications=password operator@<ip>   # refusé
ssh-audit <ip>                                 # aucun algorithme faible
```

## Tests

`molecule test` monte un conteneur Incus Ubuntu 24.04, joue le rôle deux fois (la seconde
sans changement) et vérifie SSH, ufw et CrowdSec. `molecule/default/molecule.yml`.
