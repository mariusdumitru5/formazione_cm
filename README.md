<h1 align="center">Configuration Management</h1>

Questa repo contiene tutto quello che è stato richiesto nei vari step della Track 3. 

### Step 1 - Creare il Primo Playbook

Questo step prevede la creazione di un playbook ansible che configuri un docker registry (anche senza autenticazione). 

Per fare questo primo step è stato creato il file `playbooks/container-playbook.yaml`. Il tutto è stato testato su una VM creata con Vagrant.

##### container-playbook.yaml

Questo playbook si occupa di installare docker(tramite un role sviluppato in precedenza) sulla macchina target(Rocky Linux 9). 

```yaml
roles:
    - role: install_docker
```

Dopo aver fatto questo, dato che siamo su Rocky 9, dobbiamo aprire anche la porta attraverso la quale comunichiamo con il registry(` registry_port: "5000"`).


```yaml
 - name: Apri la porta del Registry nel firewall (firewalld)
   ansible.posix.firewalld:
     port: "{{ registry_port }}/tcp"
     permanent: true
     state: enabled
     immediate: true
   ignore_errors: true  
```


Per poter salvare le immagini in locale bisogna creare un directory locale:

```yaml
- name: Crea la directory locale per la persistenza delle immagini del Registry
  ansible.builtin.file:
    path: "{{ registry_storage_dir }}"
    state: directory
    mode: '0755'
```

Infine, possiamo far partire un container docker che usa l'immagine ufficiale di docker per il registry locale `registry:2`.

```yaml
- name: Avvia il container del Docker Registry (registry:2)
  community.docker.docker_container:
    name: "{{ registry_container_name }}"
    image: registry:2
    state: started
    restart_policy: always
    published_ports:
      - "{{ registry_port }}:5000"
    env:
      REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY: /var/lib/registry
    volumes:
      - "{{ registry_storage_dir }}:/var/lib/registry:z"
```

In questo caso bisogna notare stiamo facendo un `bind mount`, cioè stiamo collegando direttamente una cartella della VM ad una cartella interna al container. In questo caso viene garantita la persistenza dei dati: se il container viene riavviato, fermato o distrutto, le immagini rimarranno al sicuro sul disco del server.

Da notare anche `:z`. Questo gestisce le autorizzazioni di SELinux per il container. L'opzione `:z` (minuscola) dice a Docker o Podman di rietichettare automaticamente la cartella dell'host con un contesto di sicurezza condiviso. Senza di questo, SELinux bloccherà i permessi di scrittura del container sulla cartella del server, restituendo un errore di tipo `Permission Denied`.

Si usa `:z` (minuscola) per permettere a più container di accedere alla stessa cartella sul server, mentre `:Z` (maiuscola) rende la cartella esclusiva e privata per un singolo container.

### Step 2 - Creare Build di container

Utilizzando Ansible, creare dei playbooks che facciano la build di almeno due container con OS differenti.

Queste build devono generare dei container che abbiano queste caratteristiche:

- Essere sempre in ascolto sulla porta 22 del container
- Avere attivo il servizio ssh
- Avere un utente abilitato a collegarsi tramite ssh key e poter fare sudo

Per fare questo step è stato creato il playbook `playbooks/build-containers.yaml` e due Dockerfile: `Dockerfiles/Dockerfile.ubuntu` e `Dockerfiles/Dockerfile.rocky`. Il tutto è stato testato in locale. 

##### Dockerfile.ubuntu

Questo Dockerfile utilizza `ubuntu:22.04` come immagine base e configura un ambiente accessibile tramite SSH per Ansible:

1. Viene creato l'utente `ansible` con privilegi `sudo` senza richiesta di password:

```Dockerfile
RUN useradd -m -s /bin/bash ansible && \
    echo "ansible ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/ansible && \
    chmod 0440 /etc/sudoers.d/ansible
```

- `useradd -m -s /bin/bash ansible`: Crea l'utente di sistema ansible, genera la sua home directory (/home/ansible) e imposta Bash come shell predefinita.
- `NOPASSWD:ALL in /etc/sudoers.d/ansible`: Garantisce all'utente privilegi di root senza richiedere l'inserimento interattivo della password. Questo è fondamentale per le esecuzioni automatiche non presidiate dei playbook Ansible.
- `chmod 0440`: Imposta il file in sola lettura per il proprietario e per il gruppo root. Per ragioni di sicurezza, `sudo` ignora e rifiuta automaticamente qualsiasi file in `/etc/sudoers.d/` che possegga permessi troppo aperti.

2. Tramite l'argomento di build `SSH_PUBLIC_KEY`, viene iniettata la chiave pubblica necessaria all'autenticazione:

```Dockerfile
ARG SSH_PUBLIC_KEY
RUN mkdir -p /home/ansible/.ssh && chmod 700 /home/ansible/.ssh && \
    echo "$SSH_PUBLIC_KEY" > /home/ansible/.ssh/authorized_keys && \
    chmod 600 /home/ansible/.ssh/authorized_keys && \
    chown -R ansible:ansible /home/ansible/.ssh
```

- `chmod 700` su `.ssh`: Limita i permessi della cartella alla sola lettura, scrittura ed esecuzione per l'utente ansible `(rwx------)`.
- `chmod 600` su `authorized_keys`: Imposta l'accesso al file contenente la chiave pubblica in sola lettura e scrittura per il proprietario `(rw-------)`.
- `chown -R ansible:ansible`: Assegna la proprietà di directory e file all'utente ansible.
  
`OpenSSH` abilita di default la direttiva `StrictModes yes`. Se la cartella `.ssh` o il file `authorized_keys` hanno permessi di gruppo o globali troppo permissivi (es. 777 o 644), il server SSH rifiuterà silenziosamente qualsiasi tentativo di autenticazione.

3. Vengono applicate le direttive di sicurezza sul `daemon sshd` (disabilitazione del login da root e via password, restrizione degli accessi al solo utente ansible):

```Dockerfile
RUN mkdir /var/run/sshd && \
    sed -i 's/#PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config && \
    sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config && \
    sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config && \
    echo 'AllowUsers ansible' >> /etc/ssh/sshd_config
```

- `mkdir /var/run/sshd`: Crea la cartella di runtime usata da `sshd` per la separazione dei privilegi. Nei container Ubuntu la mancanza di questa directory impedisce al daemon SSH di avviarsi.
- `PermitRootLogin no`: Disabilita l'accesso diretto SSH come utente root, riducendo la superficie d'attacco.
- `PasswordAuthentication no`: Disattiva completamente l'autenticazione tramite password, rendendo obbligatorio l'uso delle chiavi SSH e proteggendo il container da attacchi brute-force.
- `AllowUsers ansible`: Applica una restrizione esplicita che autorizza esclusivamente l'utente ansible a stabilire sessioni SSH.

4. Viene dichiarata la `porta 22` e configurato l'avvio automatico del server SSH in foreground:

```Dockerfile
EXPOSE 22
CMD ["/usr/sbin/sshd", "-D"]
```

Per quanto riguarda `Dockerfile.rocky` il procedimento è quasi identico. Bisogna fare `RUN ssh-keygen -A` dopo aver installato il server ssh perché l'installazione del pacchetto `openssh-server` non crea automaticamente le chiavi host del server SSH. Quindi il comando serve proprio a generare tutte le chiavi host mancanti per SSH.

##### build-containers.yaml

1. Vengono generati le chiavi ssh tramite il modulo ` community.crypto.openssh_keypair` e poi viene salvato il contenuto della chiave pubblica nella variabile `pub_key_content`:

```yaml
- name: Genera la coppia di chiavi SSH 
  community.crypto.openssh_keypair:
    path: "{{ playbook_dir }}/../Dockerfiles/id_rsa"
    type: rsa
    state: present
  register: ssh_key_result

- name: Leggo contenuto della chiave
  ansible.builtin.slurp:
    src: "{{ playbook_dir }}/../Dockerfiles/id_rsa.pub"
  register: pub_key_encoded

- name: Imposto la variabile con il testo della chiave pubblica
  ansible.builtin.set_fact:
    pub_key_content: "{{ pub_key_encoded.content | b64decode }}"
```

2. Vengono buildate le due immagini docker e avviati i container corrispettivi attraverso i moduli `community.docker.docker_image` e `community.docker.docker_container`:

```yaml
- name: Build immagine Ubuntu 22.04
  community.docker.docker_image:
    name: ubuntu-ssh-node
    build:
      path: ./Dockerfiles
      dockerfile: Dockerfile.ubuntu
      args:
        SSH_PUBLIC_KEY: "{{ pub_key_content }}"
      pull: true
    source: build

- name: Avvio container Ubuntu (Porta 2221)
  community.docker.docker_container:
    name: node-ubuntu
    image: ubuntu-ssh-node
    state: started
    restart_policy: always
    ports:
      - "2221:22"
```

### Step 3 - Creazione di un ruolo

- Utilizzando i precedenti Step come task, crea più ruoli Ansibile con le seguenti caratteristiche:
  - Creazione e configurazione di un registry 
  - Build di almeno due container 
  - Push delle build sul registry precedentemente creato
  - Run dei container in modo che non vadano in conflitto di porte tra loro
- Sarà considerato un plus se tutto parametrizziato
- Creare uno o più ruoli che funzionino sia con Docker che con Podman 

Per questo step è bastato creare i role con `ansible-galaxy role init` e poi adattare i playbook scritti prima. 

La parte più interessante è stata l'ultimo punto in cui è stato creato un ruolo che funziona sia con Docker che con Podman, in base a quello che c'è sulla macchina taget. Il ruolo in questione si chiama `roles/generic_engine_build`. 

#### generic_engine_build

1. `defaults/main.yaml`
   
Questo file contiene le variabili che possono essere facilmente sovrascrivibili. In particolare si trova il nome dell'immagine da buildare, il nome del container e l'engine che deve essere usato per la build dell'immagine e la gestione del container:

```yaml
# 'auto', 'docker' o 'podman'
container_engine: "auto"

# Configurazione del container
container_image: "immagine-ubuntu:1.0"
dockerfile_name: Dockerfile.ubuntu
```

2. `tasks/main.yaml`
   
La task principale del role. Verifica ce c'è installato Podman sulla macchina host e salva lo stato. In base a questo viene impostata la variabile `effective_engine` che serve poi a includere un altro file di task.

```yaml
- name: Imposta l'engine effettivo da utilizzare
  ansible.builtin.set_fact:
    effective_engine: >-
      {{
        'podman' if (container_engine == 'auto' and podman_check.rc | default(1) == 0)
        else (container_engine if container_engine != 'auto' else 'docker')
      }}
```

Se sul sistema c'è Podman(ed è stato scelto come engine) oppure l'engine è impostato su `auto`, allora `effective_engine = podman`. Altrimenti `effective_engine = docker`:

```yaml
- name: Esecuzione task per {{ effective_engine }}
  ansible.builtin.include_tasks: "engine_{{ effective_engine }}.yaml"
```
In questo modo viene scelto il file giusto con le task relative all'engine scelto.

3. `tasks/engine_docker.yaml`

Fa la build dell'immagine con Docker 

```yaml
- name: Build dell'immagine container tramite Docker
  community.docker.docker_image:
    name: "{{ container_image }}"
    build:
      path: "./Dockerfiles"          
      dockerfile: "{{ dockerfile_name | default('Dockerfile') }}"
      pull: true  
    source: build
    state: present
```

4. `tasks/engine_podman.yaml`

Fa la build dell'immagine con Podman.

```yaml
- name: Build dell'immagine container tramite Podman
  containers.podman.podman_image:
    name: "{{ container_image }}"
    path: "./Dockerfiles"         
    dockerfile: "{{ dockerfile_name | default('Dockerfile') }}" 
    build:
      cache: true
      extra_args: "--no-cache"     
    state: build
```

### Step 4 - Vault

Per questo step non è stato fatto nulla di particolare perché non ci sono password o informazioni sensibili nei vari playbook fatti. 

##### `Ansible Vault`

Ansible Vault è una funzionalità nativa di Ansible che permette di crittografare file e variabili sensibili (come password, chiavi SSH, ...) direttamente all'interno del progetto. 

Per creare un file crittografato: 

```bash
ansible-vault create secrets.yaml
```

Per modificare un file esistente:

```bash
ansible-vault edit secrets.yaml
```

Per crittografare un file esistente 

```bash
ansible-vault encrypt secrets.yaml
```

Per decrittografare un file:

```bash
ansible-vault decrypt secrets.yaml
```

Per visualizzare il contenuto in chiaro e basta:

```bash
ansible-vault view secrets.yaml
```

Per integrare i valori contenuti nel vault all'interno di un playbook basta includere il file:

```yaml
vars_files:
    - secrets.yaml 
```

Per eseguire un playbook che contiene valori all'interno del vault è necessario inserire la password creata quando si criptano i dati:

```bash
# chiede la password a terminale in modo interattivo
ansible-playbook playbook.yml --ask-vault-pass

# file di testo locale contenente la password
ansible-playbook playbook.yml --vault-password-file .vault_pass
```

###### Esempio utilizzo

1. Prendo un file che contiene le password di due utenti:

```yaml
# secrets.yaml
user_1_pass: 1234
user_2_pass: 5678
```

2. Cripto il file con `ansible-vault`:

```bash
ansible-vault encrypt secrets.yaml
```

3. Utilizzo le password in un file dove definisco gli utenti:

```yaml
---
# users.yaml
users:
  marius:
    state: present 
    groups: sudo
    append: true
    home: /home/marius 
    shell: /bin/bash
    password: "{{ user_1_pass }}" # variabile che si trova nel vault
    update_password: always
  tom:
    state: present 
    groups: sudo
    append: true
    home: /home/tom
    shell: /bin/bash
    password: "{{ user_2_pass }}" # variabile che si trova nel vault
    update_password: always
```

##### Nota: 

Dato che la sintassi `{{ user_1_pass }}` prende il valore dal vault e lo mette in chiaro, per utilizzare la password come una password criptata devo usare un filtro `jinja2` per criprare il testo!

4. Utilizzo gli utenti in un playbook:

```yaml
---
- name: Gestione utenti
    ansible.builtin.user:
      name: "{{  item.key }}"
      state: "{{ item.value.state }}"
      groups: "{{ item.value.groups }}"
      append: "{{ item.value.append }}"
      home: "{{ item.value.home }}"
      shell: "{{ item.value.shell }}"
      password: "{{ item.value.password | password_hash('sha512')}}"
      update_password: "{{ item.value.update_password }}"
    loop: "{{ users | dict2items }}"
```

### Step 5 - Jenkins & Ansible

- Creare un container che oltre ai requisiti dello Step 2 abbia anche le seguenti caratteristiche:
  - Avere attivo il servizio Docker/Podman;
- Configurare una pipeline Jenkins che:
  - Esegua una build di un'immagine e la tagghi in modo progressivo
  - Faccia il push dell'immagine sul registry
  - Utilizzi Ansibile per eseguire il deploy sul container precedentemente creato 
  
1. Per aggiungere docker ad uno dei container fatti allo step 2 basta modificare il Dockerfile. In particolare è stato creato un nuovo Dockerfile chiamato `Dockerfiles/Dockerfile.docker`. Si tratta di un caso di `Docker in Docker`. Quello che va aggiunto è:

```Dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release \
    openssh-server \
    sudo \
    iptables \
    && rm -rf /var/lib/apt/lists/*
```

Sono stati aggiunti i pacchetti necessari affinché Docker funzioni.

```Dockerfile
RUN mkdir -p /etc/apt/keyrings && \
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg && \
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu jammy stable" > /etc/apt/sources.list.d/docker.list && \
    apt-get update && apt-get install -y --no-install-recommends \
    docker-ce \
    docker-ce-cli \
    containerd.io \
    && rm -rf /var/lib/apt/lists/*
```

- `curl -fsSL ...`: Scarica la chiave di cifratura ufficiale di Docker in modo "silenzioso" e seguendo eventuali re-indirizzamenti.
- `gpg --dearmor`: Converte la chiave dal formato testo (ASCII-armored) al formato binario compresso richiesto da apt.
- `-o /etc/apt/...`: Salva la chiave convertita nel file docker.gpg. Serve a apt per verificare che i pacchetti Docker che scaricherai siano autentici e non manomessi.
- `arch=$(dpkg --print-architecture)`: Rileva automaticamente l'architettura del sistema (es. amd64 per Intel/AMD o arm64 per Apple Silicon/ARM).
- `signed-by=...`: Dice ad apt di fidarsi del repository solo se firmato con la chiave GPG salvata al punto 2.
- `jammy stable`: Specifica la versione di Ubuntu (jammy = 22.04) e il ramo di pacchetti stabili.

```bash
usermod -aG docker ansible && \
```

L'utente `ansible` è stato aggiunto al gruppo `sudo`. 

```Dockerfile
CMD service docker start && /usr/sbin/sshd -D
```

Avvia il docker engine e il server ssh.