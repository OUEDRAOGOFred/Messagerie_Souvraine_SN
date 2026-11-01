# Partie A — Messagerie chiffrée

Cette partie fournit Synapse, PostgreSQL et Element. Elle permet aux agents
d'échanger des messages, fichiers et messages vocaux chiffrés de bout en bout.

## Démarrage autonome

```bash
cp .env.example .env
docker compose config -q
docker compose up -d caddy synapse synapse-db element
```

En développement autonome, `SYNAPSE_PASSWORD_LOGIN=true` permet de créer des
comptes de test. Cette option devra être remise à `false` après l'intégration
de Keycloak.

Créer les comptes de démonstration autonomes :

```bash
chmod +x messagerie/scripts/*.sh
messagerie/scripts/create-demo-users.sh
```

Vérifier les routes et la protection de l'API d'administration :

```bash
messagerie/scripts/verify-part-a.sh
```

Tester une connexion Matrix autonome :

```bash
curl -X POST http://matrix.localhost/_matrix/client/v3/login \
  -H 'Content-Type: application/json' \
  -d '{
    "type": "m.login.password",
    "identifier": {"type": "m.id.user", "user": "awa.test"},
    "password": "motdepasse-test"
  }'
```

La preuve du chiffrement doit être faite avec Element : les événements de
messages, fichiers et vocaux doivent être envoyés depuis des salons chiffrés.
Dans PostgreSQL, les événements attendus sont alors principalement de type
`m.room.encrypted`; le serveur ne possède pas les clés privées nécessaires au
déchiffrement. Après avoir envoyé un message dans Element, inspecter les types
stockés :

```bash
docker compose --env-file .env.example exec synapse-db psql \
  -U synapse -d synapse \
  -c 'SELECT type, count(*) FROM events GROUP BY type ORDER BY count(*) DESC;'
```

Les événements de contenu doivent apparaître sous `m.room.encrypted`, et non
comme des événements `m.room.message` lisibles.

## Contrats d'intégration

- C1 : Synapse utilise le realm Keycloak `administration`, le client `synapse`
  et le groupe `agents`.
- C2 : la console de la partie B utilise Synapse sur `http://synapse:8008`.
  Les chemins `/_synapse/admin/*` répondent toujours `403` via Caddy.
- C3 : Element est servi par `chat.<DOMAIN>` et Matrix par `matrix.<DOMAIN>`.
