Terraform-GKE-Vault-Consul-Postgres Demo

Description: <br>
This demo is designed to show how simple it is to deploy a kubernetes cluster in GCP using Terraform Cloud. It will then deploy Consul and Vault on the same cluster and finally, use Vault to generate dynamic secrets for a Postgres database.

To view the YouTube videos please click on the below links: <br>

Part 1 - https://youtu.be/BOQG9iaHA24

Part 2 - 

Part 3 - 

This repo will contain all the code used in the demo so please watch the videos to learn when and where to use which line of code.

gcloud container clusters get-credentials k8s --zone asia-southeast1-a --project hashi-kubernetes-db

Open new terminal for each port forward:

kubectl port-forward -n postgres consul-server-0 8080:8500 <br>
kubectl port-forward -n postgres consul-db-vault-0 8081:8200 <br>

kubectl get pods -n postgres

kubectl exec -it -n postgres consul-db-vault-0 -- vault operator init

Unseal Key 1: <br>
Unseal Key 2: <br>
Unseal Key 3: <br>
Unseal Key 4: <br>
Unseal Key 5: <br>

Initial Root Token: 

Unseal the first vault server until it reaches the key threshold <br>
kubectl exec --stdin=true --tty=true -n postgres consul-db-vault-0 -- vault operator unseal (unseal key 1) <br>
kubectl exec --stdin=true --tty=true -n postgres consul-db-vault-0 -- vault operator unseal (unseal key 2) <br>
kubectl exec --stdin=true --tty=true -n postgres consul-db-vault-0 -- vault operator unseal (unseal key 3) <br>

kubectl exec --stdin=true --tty=true -n postgres consul-db-vault-1 -- vault operator unseal (unseal key 1) <br>
kubectl exec --stdin=true --tty=true -n postgres consul-db-vault-1 -- vault operator unseal (unseal key 2) <br>
kubectl exec --stdin=true --tty=true -n postgres consul-db-vault-1 -- vault operator unseal (unseal key 3) <br>

kubectl exec --stdin=true --tty=true -n postgres consul-db-vault-2 -- vault operator unseal (unseal key 1) <br>
kubectl exec --stdin=true --tty=true -n postgres consul-db-vault-2 -- vault operator unseal (unseal key 2) <br>
kubectl exec --stdin=true --tty=true -n postgres consul-db-vault-2 -- vault operator unseal (unseal key 3) <br>
	
kubectl exec -it -n postgres consul-db-vault-0 -- vault status <br>
kubectl exec -it -n postgres consul-server-0 -- consul members <br>

kubectl exec -it -n postgres consul-db-vault-0 -- vault login (root token)

kubectl exec -it -n postgres consul-db-vault-0 -- vault secrets enable database

kubectl exec -i -n postgres consul-db-vault-0 -- vault write postgres-policy - << EOF 
path "database/roles/*" { capabilities = ["create", "read", "update", "delete", "list"] }

path "database/config/*" { capabilities = ["create", "read", "update", "delete", "list"] }

path "database/creds/jonos_db" { capabilities = ["read"] }
EOF <br>

kubectl exec -it -n postgres consul-db-vault-0 -- vault policy list

kubectl exec -it -n postgres consul-db-vault-0 -- vault policy read postgres-policy

kubectl exec -it -n postgres consul-db-vault-0 -- vault token create -policy postgres-policy

wget https://raw.githubusercontent.com/asvesj/terraform-gke-vault-consul-postgres/master/postgres.yml

kubectl create -n postgres -f postgres.yml 

kubectl exec -it -n postgres consul-db-vault-0 -- vault write database/config/jonos_db \
    plugin_name=postgresql-database-plugin \
    allowed_roles="scorpion,subzero" \
    connection_url="postgresql://{{username}}:{{password}}@postgres:5432/jonos_db?sslmode=disable" \
    username="postgres" \
    password="password"

kubectl exec -it -n postgres consul-db-vault-0 -- vault write database/roles/scorpion  \
    db_name=jonos_db \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
    GRANT INSERT ON ALL TABLES IN SCHEMA public TO \"{{name}}\"; \
    GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    default_ttl="1h" \
    max_ttl="24h"

kubectl exec -it -n postgres consul-db-vault-0 -- vault write database/roles/subzero \
    db_name=jonos_db \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
    GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    revocation_statements="SELECT revoke_access('{{name}}'); DROP user \"{{name}}\";"\
    default_ttl="1h" \
    max_ttl="24h"

kubectl exec -it -n postgres consul-db-vault-0 -- vault write --force database/rotate-root/jonos_db

kubectl exec -it -n postgres $(kubectl get pods -n postgres --selector "app=postgres" -o jsonpath="{.items[0].metadata.name}") -c postgres -- bash -c 'PGPASSWORD=password psql -U postgres jonos_db'

kubectl exec -it -n postgres consul-db-vault-0 -- vault read database/creds/scorpion

kubectl exec -it -n postgres consul-db-vault-0 -- vault read database/creds/subzero
	
POD='kubectl get pods -n postgres -l app=postgres -o wide | grep -v NAME | awk '{print $1}''
  	
kubectl exec -it -n postgres $POD -- psql -U  jonos_db

CREATE TABLE fatality (id serial, title text);

INSERT INTO fatality (title) VALUES ('Chain Reaction');

INSERT INTO fatality (title) VALUES ('Ice-Cutioner');

INSERT INTO fatality (title) VALUES ('You are Next');

\c – to connect to db

\l – to list dbs

\du – to list roles

\dt – to list tables

\z fatality – to see table privileges

table fatality;
