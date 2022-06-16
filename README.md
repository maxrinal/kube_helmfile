# Minikube bootstrapeado con helmfile

El objetivo de este proyecto es deplegar un cluster de minikube con herramientas minimas para hacer pruebas


* metallb: Tener ips de load balancer
* ingress-nginx: desplegado usando loadbalancer con ip de metallb
* powerdns: Para registrar dominios locales para nuestro escenario de pruebas, desplegado sobre una ip virtual de loadbalancer para poder acceder externamente
* external-dns: Registro automatico de ingress en powerdns desplegado localmente


Instalamos helmfile
```bash
curl -LO https://github.com/roboll/helmfile/releases/download/v0.144.0/helmfile_linux_amd64
sudo install helmfile_linux_amd64 /usr/local/bin/helmfile

# Para usar apply se requiere el plugin helm diff
# helmfile apply requires https://github.com/databus23/helm-diff
# helm plugin install https://github.com/databus23/helm-diff
```

Instalamos minikube
```bash
curl -LO https://github.com/kubernetes/minikube/releases/download/v1.25.2/minikube-linux-amd64.tar.gz
tar xzvf minikube-linux-amd64.tar.gz
sudo install out/minikube-linux-amd64 /usr/local/bin/minikube
sudo install out/docker-machine-driver-kvm2 /usr/local/bin/docker-machine
```

Creamos un cluster de minikube


```bash
minikube start --profile mk-helm \
--driver=kvm2 --network default  \
--insecure-registry='http://192.168.122.1:4000' --registry-mirror='http://192.168.122.1:4000' \
--cpus=4 --memory=4G --cni=flannel
# Utilizamos cni=flannel ya que con el cni=bridge(default) se tienen problemas de comunicacion con servicios en el mismo ns

```

Aplicamos el helmfile

```bash
# Instalo metallb+nginx lb(192.168.122.10)+powerdns lb(192.168.122.11)+externaldns
helmfile --file helmfile.yaml sync

```

# Create Test Chart and Install

Creamos un chart de prueba con configuraciones basicas para testeo
```bash
helm create /tmp/testm

cat <<EOT | helm upgrade --install --namespace ns-test --create-namespace pepe /tmp/testm -f -
image:
  repository: ealen/echo-server
  tag: "latest"
ingress:
  enabled: true
  className: "nginx-lb"
  annotations:
    externaldns: pdns
  hosts:
    - host: lbregtls.k8s.example.test
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls:
   - secretName: tls-k8sexample
     hosts:
       - lbregtls.k8s.example.test
EOT


cat <<EOT | helm upgrade --install --namespace ns-test --create-namespace pepe2 /tmp/testm -f -
image:
  repository: nginx
  tag: alpine                                           
ingress:
  enabled: true
  className: "nginx-lb"
  annotations: 
    externaldns: pdns
    nginx.ingress.kubernetes.io/canary: "true"         # Enable canary.
    nginx.ingress.kubernetes.io/canary-weight: "50"    # Forward 20% of the traffic to the canary ingress.
  hosts:
    - host: lbregtls.k8s.example.test
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: 
   - secretName: tls-k8sexample
     hosts:
       - lbregtls.k8s.example.test
EOT


# Creamos tls secret referenciando a un crt autofirmado
# kubectl create secret tls tls-k8sexample --dry-run=client --cert=signed_cert/k8s.example.test/20230507.pem --key=signed_cert/k8s.example.test/20230507.key
# kubectl -n ns-test apply -f tmp/tlsk8sexample.yaml

# Prueba de ingress y dns
curl -k https://lbregtls.k8s.example.test

# Validamos los certificados expuestos por el servidor
openssl s_client -showcerts -connect lbregtls.k8s.example.test:443 </dev/null
```






# ADD DNS TO NETWORK MANAGER(DNSMASQ)

```bash
LIBVIRT_DNSMASQ_FILE=/etc/NetworkManager/dnsmasq.d/libvirt.conf


PATTERN_TO_SEARCH='^server=\/k8s.example.test.*$'
# LINE_TO_ADD="server=\/k8s.example.test\/192.168.122.11"
LINE_TO_ADD="server=/k8s.example.test/192.168.122.11"
LINE_TO_ADD_ESCAPED=$(echo $LINE_TO_ADD | sed 's/\//\\\//g' )


# Si no existe lo agrego
grep -q ${PATTERN_TO_SEARCH}  ${LIBVIRT_DNSMASQ_FILE} || echo "${LINE_TO_ADD}" | sudo tee -a  ${LIBVIRT_DNSMASQ_FILE}
sudo sed -i  "s/${PATTERN_TO_SEARCH}/${LINE_TO_ADD_ESCAPED}/g" ${LIBVIRT_DNSMASQ_FILE}

cat <<EOT | tee /etc/NetworkManager/conf.d/localdns.conf
[main]
dns=dnsmasq
EOT

# (resolv.conf -> ../run/systemd/resolve/stub-resolv.conf)
# mv /etc/resolv.conf /etc/resolv.conf.original_symlink

# Disabled systemd-resvoled
# https://medium.com/infraspeak/stop-messing-with-the-hosts-file-a760aa660c20
# https://trytonvanmeer.dev/posts/networkmanager-dnsmasq-libvirtd/
# https://kifarunix.com/configure-local-dns-server-using-dnsmasq-on-ubuntu-20-04/



# Recargamos dnsmasq
# sudo systemctl reload NetworkManager.service
sudo systemctl restart NetworkManager.service
```

Validamos que luego de agregar la zona a dnsmasq podemos resolver directamente
```bash
nslookup lbreg.k8s.example.test
Server:		127.0.1.1
Address:	127.0.1.1#53

Name:	lbreg.k8s.example.test
Address: 192.168.122.10
```


## DOCKER REGISTRY CACHE

Se asume disponible una docker-registry cache, para acelerar el despligue
```bash
# Creacion de volume
docker volume create cache-docker-reg

# Despligue docker-registry cache en puerto 4000 del host
docker run -d -p 4000:5000 \
    -e REGISTRY_PROXY_REMOTEURL=https://registry-1.docker.io \
    --volume cache-docker-reg:/var/lib/registry \
    --restart always \
    --name registry-docker.io registry:2

# Validacion de logs
docker logs -f registry-docker.io

# Validacion de cache de imagenes descargadas

curl localhost:4000/v2/_catalog | jq
```
