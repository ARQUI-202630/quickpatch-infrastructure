# Ansible en contenedor: misma versión para todo el equipo, sin instalar nada en el sistema.
FROM python:3.13-slim
RUN apt-get update \
 && apt-get install -y --no-install-recommends openssh-client sshpass \
 && rm -rf /var/lib/apt/lists/*
RUN pip install --no-cache-dir "ansible-core==2.18.*"
COPY requirements.yml /tmp/requirements.yml
RUN ansible-galaxy collection install -r /tmp/requirements.yml -p /usr/share/ansible/collections
WORKDIR /ansible
ENTRYPOINT ["ansible-playbook"]
