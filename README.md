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

2. Tramite l'argomento di build `SSH_PUBLIC_KEY`, viene iniettata la chiave pubblica necessaria all'autenticazione:

```Dockerfile
ARG SSH_PUBLIC_KEY
RUN mkdir -p /home/ansible/.ssh && chmod 700 /home/ansible/.ssh && \
    echo "$SSH_PUBLIC_KEY" > /home/ansible/.ssh/authorized_keys && \
    chmod 600 /home/ansible/.ssh/authorized_keys && \
    chown -R ansible:ansible /home/ansible/.ssh
```

3. Vengono applicate le direttive di sicurezza sul `daemon sshd` (disabilitazione del login da root e via password, restrizione degli accessi al solo utente ansible):

```Dockerfile
RUN mkdir /var/run/sshd && \
    sed -i 's/#PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config && \
    sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config && \
    sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config && \
    echo 'AllowUsers ansible' >> /etc/ssh/sshd_config
```

4. Viene dichiarata la `porta 22` e configurato l'avvio automatico del server SSH in foreground:

```Dockerfile
EXPOSE 22
CMD ["/usr/sbin/sshd", "-D"]
```

##### build-containers.yaml

D