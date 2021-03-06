# rancher-on-fedora-coreos - Declarative (Fedora Core OS , local vsphere)
A repo containing a YAML template for deploying Rancher on Fedora CoreOS

## Fedora Core OS
Fedora CoreOS is an automatically updating, minimal, monolithic, container-focused operating system, designed for clusters but also operable standalone, optimized for Kubernetes but also great without it. It aims to combine the best of both CoreOS Container Linux and Fedora Atomic Host, integrating technology like Ignition from Container Linux with rpm-ostree and SELinux hardening from Project Atomic. Its goal is to provide the best container host to run containerized workloads securely and at scale.

## What are we trying to do?
Copy and paste a base64 signature that has all of the instructions to setup your server with minimal setup. Deploying Kubernetes in less than 15 minutes.

to generate password_hash use and paste it in `declarative.yaml` file after `password_hash:`
``` 
openssl passwd -1
```

to generate ssh_authorized_keys
```
ssh-keygen
```
Copy the .pub key and paste it in `declarative.yaml` file after `ssh_authorized_keys:`
```
cat ${HOME}/.ssh/id_rsa.pub
```


export and convert file for vsphere
```sh

nerdctl run --rm --tty --interactive 
           --security-opt label=disable        
           --volume ${PWD}:/pwd --workdir /pwd 
             quay.io/coreos/butane:release 
           --strict declarative.yaml > declarative.json
cat declarative.json | base64 -w0 - > declarative.txt

```

vsphere - Ignition config data
```
{BASE64 CONTENT}
```

vsphere - Ignition config data encoding
```
base64
```

```yaml
variant: fcos
version: 1.4.0
# Name: Rancher
# Comment: inputs -> you need your own name, password_hash, ssh_authorized_keys, address1, dns, hostname

passwd:
  users:
    - name: {USERNAME}
      groups: 
        - docker
        - wheel
        - sudo
      password_hash: {HASH}
      ssh_authorized_keys:
        - ssh-rsa {KEY} {LOGIN}
storage:
  files:
    - path: /etc/sysctl.d/20-silence-audit.conf
      contents:
        inline: |
          kernel.printk=4
    - path: /etc/zincati/config.d/55-updates-strategy.toml
      contents:
        inline: |
          [updates]
          strategy = "periodic"
          [[updates.periodic.window]]
          days = [ "Sat", "Sun" ]
          start_time = "22:30"
          length_minutes = 60
    - path: /etc/NetworkManager/system-connections/ens192.nmconnection
      mode: 0600
      contents:
        inline: |
          [connection]
          id=ens192
          type=ethernet
          interface-name=ens192
          [ipv4]
          address1={IP/SUB}, {IP} 
          # e.g. address1=156.124.92.79/23,156.124.92.240
          dns={DNS IP ADDRESS};
          dns-search=
          may-fail=false
          method=manual
    - path: /etc/hostname
      mode: 0644
      contents:
        inline: {FQDN}
    - path: /etc/ssh/sshd_config.d/20-enable-passwords.conf
      mode: 0644
      contents:
        inline: |
          PasswordAuthentication yes
    - path: /etc/pki/ca-trust/source/anchors/root.crt
      contents:
        source: http://{FQDNTOCERTS}/CertEnroll/root.crt
    - path: /etc/pki/ca-trust/source/anchors/Intermediate_East.crt
      contents:
        source: http://{FQDNTOCERTS}/CertEnroll/Intermediate_East.crt
    - path: /etc/pki/ca-trust/source/anchors/Intermediate_West.crt
      contents:
        source: http://{FQDNTOCERTS}/CertEnroll/Intermediate_West.crt
    - path: /etc/pki/ca-trust/source/anchors/Device_Intermediate_East.crt
      contents:
        source: http://{FQDNTOCERTS}/CertEnroll/Device_Intermediate_East.crt
    - path: /etc/pki/ca-trust/source/anchors/Device_Intermediate_West.crt
      contents:
        source: http://{FQDNTOCERTS}/CertEnroll/Device_Intermediate_West.crt
    - path: /usr/local/bin/setup-certs
      mode: 0755
      contents:
        inline: |
          #!/usr/bin/env sh
          main() {
            /bin/update-ca-trust
            return 0
          }
          main
    - path: /usr/local/bin/run-k3s-prereq-installer
      mode: 0755
      contents:
        inline: |
          #!/usr/bin/env sh
          main() {
            rpm-ostree install https://github.com/k3s-io/k3s-selinux/releases/download/v0.5.stable.1/k3s-selinux-0.5-1.el8.noarch.rpm
            return 0
          }
          main
    - path: /usr/local/bin/run-k3s-installer
      mode: 0755
      contents:
        inline: |
          #!/usr/bin/env sh
          main() {
            export K3S_KUBECONFIG_MODE="644"
            # export INSTALL_K3S_EXEC=" --no-deploy servicelb --no-deploy traefik"
            export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
            curl -sfL https://get.k3s.io | sh -
            return 0
          }
          main
    - path: /usr/local/bin/run-helm-installer
      mode: 0755
      contents:
        inline: |
          #!/usr/bin/env sh
          main() {
            export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
            # curl -sfL https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | sh -
            curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
            return 0
          }
          main
    - path: /usr/local/bin/run-rancher-installer
      mode: 0755
      contents:
        inline: |
          #!/usr/bin/env sh
          main() {
            export KUBECONFIG=/var/home/{USERNAME}/.kube/config
            export HELM_CACHE_HOME=/var/home/{USERNAME}/.cache/helm
            export HELM_CONFIG_HOME=/var/home/{USERNAME}/.config/helm
            export HELM_DATA_HOME=/var/home/{USERNAME}/.local/share/helm
            export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
            helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
            helm repo add jetstack https://charts.jetstack.io
            read -t 5 -p "I am going to wait for 5 seconds then try a helm repo update ..."
            helm repo update
            read -t 5 -p "I am going to wait for 5 seconds then try to install cert manager ..."
            helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.6.1 --set installCRDs=true
            read -t 5 -p "I am going to wait for 5 seconds then try to install rancher ..."
            helm install rancher rancher-stable/rancher --namespace cattle-system --create-namespace --set bootstrapPassword={BOOTSTRAPPASSWORD} --set hostname=${HOSTNAME} --set ingress.tls.source=rancher
            return 0
          }
          main
    - path: /home/{USERNAME}/.config/systemd/user/run-rancher-installer.service
      mode: 0644
      contents:
        inline: |
          [Unit]
          After=network-online.target
          Wants=network-online.target
          Before=systemd-user-sessions.service
          OnFailure=emergency.target
          OnFailureJobMode=replace-irreversibly
          ConditionPathExists=/var/lib/helm-installed
          ConditionPathExists=!/home/{USERNAME}/rancher-installed
          [Service]
          RemainAfterExit=yes
          Type=oneshot
          ExecStart=/usr/local/bin/run-rancher-installer
          ExecStartPost=/usr/bin/touch /home/{USERNAME}/rancher-installed
          StandardOutput=kmsg+console
          StandardError=kmsg+console
          [Install]
          WantedBy=multi-user.target
      user:
        name: {USERNAME}
      group:
        name: {USERNAME}
    - path: /var/lib/systemd/linger/{USERNAME}
      mode: 0644
  directories:
    - path: /home/{USERNAME}/.config
      user:
        name: {USERNAME}
      group:
        name: {USERNAME}
    - path: /home/{USERNAME}/.config/systemd
      user:
        name: {USERNAME}
      group:
        name: {USERNAME}
    - path: /home/{USERNAME}/.config/systemd/user
      user:
        name: {USERNAME}
      group:
        name: {USERNAME}
    - path: /home/{USERNAME}/.config/systemd/user/default.target.wants
      mode: 0755
      user:
        name: {USERNAME}
      group:
        name: {USERNAME}
  links:
    - path: /home/{USERNAME}/.config/systemd/user/default.target.wants/run-rancher-installer.service
      user:
        name: {USERNAME}
      group:
        name: {USERNAME}
      target: /home/{USERNAME}/.config/systemd/user/run-rancher-installer.service
      hard: false
systemd:
  units:
    - name: hello.service
      enabled: true
      contents: |
        [Unit]
        Description=A hello world unit!
        [Service]
        Type=oneshot
        RemainAfterExit=yes
        ExecStart=/usr/bin/echo "Hello, World!"
        [Install]
        WantedBy=multi-user.target
    - name: setup-certs.service
      enabled: true
      contents: |
        [Unit]
        After=network-online.target
        Wants=network-online.target
        Before=systemd-user-sessions.service
        OnFailure=emergency.target
        OnFailureJobMode=replace-irreversibly
        ConditionPathExists=/etc/pki/ca-trust/source/anchors/root.crt
        [Service]
        RemainAfterExit=yes
        Type=oneshot
        ExecStart=/usr/local/bin/setup-certs
        ExecStartPost=/usr/bin/touch /var/lib/setup-certs-installed
        StandardOutput=kmsg+console
        StandardError=kmsg+console
        [Install]
        WantedBy=multi-user.target
    - name: run-k3s-prereq-installer.service
      enabled: true
      contents: |
        [Unit]
        After=network-online.target
        Wants=network-online.target
        Before=systemd-user-sessions.service
        OnFailure=emergency.target
        OnFailureJobMode=replace-irreversibly
        ConditionPathExists=!/var/lib/k3s-prereq-installed
        [Service]
        RemainAfterExit=yes
        Type=oneshot
        ExecStart=/usr/local/bin/run-k3s-prereq-installer
        ExecStartPost=/usr/bin/touch /var/lib/k3s-prereq-installed
        ExecStartPost=/usr/bin/systemctl --no-block reboot
        StandardOutput=kmsg+console
        StandardError=kmsg+console
        [Install]
        WantedBy=multi-user.target
    - name: run-k3s-installer.service
      enabled: true
      contents: |
        [Unit]
        After=network-online.target
        Wants=network-online.target
        Before=systemd-user-sessions.service
        OnFailure=emergency.target
        OnFailureJobMode=replace-irreversibly
        ConditionPathExists=/var/lib/k3s-prereq-installed
        ConditionPathExists=!/var/lib/k3s-installed
        [Service]
        RemainAfterExit=yes
        Type=oneshot
        ExecStart=/usr/local/bin/run-k3s-installer
        ExecStartPost=/usr/bin/touch /var/lib/k3s-installed
        ExecStartPost=/usr/bin/systemctl --no-block reboot
        StandardOutput=kmsg+console
        StandardError=kmsg+console
        [Install]
        WantedBy=multi-user.target
    - name: run-helm-installer.service
      enabled: true
      contents: |
        [Unit]
        After=network-online.target
        Wants=network-online.target
        Before=systemd-user-sessions.service
        OnFailure=emergency.target
        OnFailureJobMode=replace-irreversibly
        ConditionPathExists=/var/lib/k3s-installed
        ConditionPathExists=!/var/lib/helm-installed
        [Service]
        RemainAfterExit=yes
        Type=oneshot
        ExecStart=/usr/local/bin/run-helm-installer
        ExecStartPost=/usr/bin/touch /var/lib/helm-installed
        ExecStartPost=/usr/bin/systemctl --no-block reboot
        StandardOutput=kmsg+console
        StandardError=kmsg+console
        [Install]
        WantedBy=multi-user.target
```
