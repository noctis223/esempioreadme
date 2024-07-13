# Makefile per l'installazione di Harbor su Kubernetes

# Variabili
NAMESPACE = default
HELM_REPO = harbor
HELM_REPO_URL = https://helm.goharbor.io
INGRESS_DOMAIN = kube-node-2.kg.kireygroup.com
DOCKER_CERT_DIR = /etc/docker
DOCKER_CERT_NAME = hostnamediharbor

# Prerequisiti
.PHONY: prerequisites
prerequisites:
	@echo "Verifica prerequisiti..."
	@which rancher || (echo "Errore: Rancher non è installato"; exit 1)
	@which kubectl || (echo "Errore: kubectl non è installato"; exit 1)
	@which helm || (echo "Errore: Helm non è installato"; exit 1)
	@kubectl version --short | grep -q "Server Version: v1.20" || (echo "Errore: Kubernetes cluster deve essere versione 1.20+"; exit 1)
	@helm version --short | grep -q "v3.2.0" || (echo "Errore: Helm deve essere versione 3.2.0+"; exit 1)
	@echo "Prerequisiti verificati."

# Aggiunta repository Helm
.PHONY: add-helm-repo
add-helm-repo:
	@echo "Aggiunta repository Helm..."
	helm repo add $(HELM_REPO) $(HELM_REPO_URL)
	helm repo update
	@echo "Repository Helm aggiunta."

# Installazione di Harbor
.PHONY: install-harbor
install-harbor: add-helm-repo
	@echo "Installazione di Harbor..."
	helm install harbor $(HELM_REPO)/harbor --namespace $(NAMESPACE)
	@echo "Harbor installato nel namespace $(NAMESPACE)."

# Modifica ingress
.PHONY: modify-ingress
modify-ingress:
	@echo "Modifica ingress..."
	kubectl get ingress -n $(NAMESPACE) -o yaml | sed 's/core\.harbor\.domain/$(INGRESS_DOMAIN)/g' | kubectl apply -f -
	@echo "Ingress modificato con dominio $(INGRESS_DOMAIN)."

# Aggiunta certificati e configurazione daemon Docker
.PHONY: configure-docker
configure-docker:
	@echo "Configurazione Docker..."
	@echo "Scarica i certificati da Harbor e posizionali in $(DOCKER_CERT_DIR)/$(DOCKER_CERT_NAME)/"
	@read -p "Premi invio dopo aver posizionato i certificati."
	@mkdir -p $(DOCKER_CERT_DIR)
	@echo '{ "insecure-registries": ["https://$(DOCKER_CERT_NAME)"] }' > $(DOCKER_CERT_DIR)/daemon.json
	sudo systemctl restart docker
	@echo "Docker configurato."

# Login a Harbor
.PHONY: login-harbor
login-harbor:
	@echo "Login a Harbor..."
	docker login $(DOCKER_CERT_NAME)
	@echo "Login effettuato."

# Verifica degli errori e soluzioni
.PHONY: check-errors
check-errors:
	@echo "Verifica errori comuni..."
	@echo "Errore timeout: Verifica che la configurazione proxy sia stata eliminata nel file YAML di Harbor."
	@echo "Errore x509: Verifica la correttezza del file daemon.json e che i certificati siano nella cartella corretta."
	@echo "Verifica errori completata."

# Target principale
.PHONY: all
all: prerequisites install-harbor modify-ingress configure-docker login-harbor check-errors
	@echo "Installazione e configurazione completate."
