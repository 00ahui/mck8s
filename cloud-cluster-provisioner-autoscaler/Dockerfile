FROM python:3.7
RUN pip3 install kopf kubernetes pint PyYAML pandas
COPY helm /usr/local/bin/helm
RUN chmod +x /usr/local/bin/helm
COPY kubectl /usr/local/bin/kubectl
RUN chmod +x /usr/local/bin/kubectl
COPY kubefedctl /usr/local/bin/kubefedctl
RUN chmod +x /usr/local/bin/kubefedctl
COPY configure_cilium_1.sh configure_cilium_2.sh ./
RUN chmod +x configure_cilium_1.sh
RUN chmod +x configure_cilium_2.sh
COPY flannel.yaml calico.yaml cilium.yaml ./
COPY prom-remote.yaml /prom-remote.yaml
COPY utils.py /utils.py
COPY cpca.py /cloud_provisioner.py
CMD kopf run --standalone /cloud_provisioner.py
