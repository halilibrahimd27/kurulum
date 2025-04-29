# Ubuntu 22.04 Üzerinde Tek Sunuculu (Single Node) Kubernetes Kurulumu

Bu dokümantasyon, tek bir fiziksel sunucu üzerinde Kubernetes kurulumunu sıfırdan detaylı bir şekilde anlatmaktadır. All-in-One olarak da bilinen bu kurulum yaklaşımında, sunucunuz hem master (control-plane) hem de worker rolünü üstlenecektir.

## Donanım Gereksinimleri

Minimum gereksinimler:
- CPU: 2 çekirdek
- RAM: 4 GB
- Disk: 20+ GB boş alan
- İnternet bağlantısı

Önerilen gereksinimler:
- CPU: 4+ çekirdek
- RAM: 8+ GB
- Disk: 50+ GB boş alan
- Sabit İnternet bağlantısı

## 1. Sistem Hazırlığı

### 1.1. Sistem Güncellemesi

İlk olarak sisteminizi güncelleyin:

```bash
sudo apt update
sudo apt upgrade -y
```

### 1.2. Hostname Ayarları

Sunucunuzun hostname'ini Kubernetes node'u olarak ayarlayın:

```bash
# Mevcut hostname'i kontrol edin
hostname

# Hostname'i değiştirin
sudo hostnamectl set-hostname k8s-node

# Hostname'in doğru şekilde ayarlandığını doğrulayın
hostname
```

### 1.3. /etc/hosts Dosyasının Düzenlenmesi

Yerel DNS çözümlemesi için hosts dosyasını düzenleyin:

```bash
# Sunucunuzun IP adresini öğrenin
ip addr show | grep inet | grep -v "127.0.0.1" | grep -v "inet6"

# Veya daha basit bir şekilde:
hostname -I
```

Çıkan IP adresini (örneğin 192.168.1.100) not alın ve hosts dosyasına ekleyin:

```bash
sudo nano /etc/hosts
```

Dosyaya şu satırı ekleyin (IP adresini kendi IP adresinizle değiştirin):

```
192.168.1.100 k8s-node
```

Düzenlemeyi kaydedip çıkın (Nano editöründe CTRL+O, ENTER, CTRL+X).

### 1.4. Swap'ı Devre Dışı Bırakma

Kubernetes'in düzgün çalışması için swap kapatılmalıdır:

```bash
# Geçici olarak swap'ı kapatın
sudo swapoff -a

# Swap'ı kalıcı olarak devre dışı bırakın
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Swap'ın kapalı olduğunu doğrulayın
free -h
```

### 1.5. Gerekli Kernel Modüllerini Yükleme

Kubernetes'in gerektirdiği kernel modüllerini yükleyin:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Modüllerin yüklendiğini doğrulayın
lsmod | grep -e overlay -e br_netfilter
```

### 1.6. Kernel Parametrelerini Ayarlama

Kubernetes için gerekli kernel parametrelerini ayarlayın:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Parametreleri uygulayın
sudo sysctl --system

# Parametrelerin uygulandığını doğrulayın
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

## 2. Container Runtime (containerd) Kurulumu

### 2.1. Gerekli Paketlerin Yüklenmesi

```bash
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common gnupg lsb-release
```

### 2.2. Docker GPG Anahtarını Ekleme

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

### 2.3. Docker Repository'sini Ekleme

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 2.4. Containerd Kurulumu

```bash
sudo apt update
sudo apt install -y containerd.io
```

### 2.5. Containerd Yapılandırması

Varsayılan yapılandırma dosyasını oluşturun:

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
```

SystemdCgroup'u etkinleştirin (Kubernetes için önemli):

```bash
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
```

Düzenlemeyi kontrol edin:

```bash
grep SystemdCgroup /etc/containerd/config.toml
```

### 2.6. Containerd'ı Yeniden Başlatma

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd
```

## 3. Kubernetes Bileşenlerinin Kurulumu

### 3.1. Kubernetes GPG Anahtarını Ekleme

```bash
sudo curl -fsSLo /etc/apt/keyrings/kubernetes-apt-keyring.gpg https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key
```

### 3.2. Kubernetes Repository'sini Ekleme

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

### 3.3. Kubernetes Paketlerini Kurma

```bash
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Kurulumu doğrulayın:

```bash
kubectl version --client
kubeadm version
```

## 4. Kubernetes Single-Node Cluster Oluşturma

### 4.1. Kubeadm ile Cluster Başlatma

Sunucunuzun IP adresini belirleyin:

```bash
IP_ADDR=$(hostname -I | awk '{print $1}')
echo $IP_ADDR
```

Bu IP adresini kullanarak cluster'ı başlatın:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=$IP_ADDR --control-plane-endpoint=$IP_ADDR
```

**Not:** Kubeadm init komutu başarılı olduğunda, çıktının sonunda kubeconfig ayarı ve node join komutu görüntülenecektir. Bu bilgileri kaydedin.

### 4.2. Kubeconfig Dosyasını Ayarlama

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Yapılandırmayı doğrulayın:

```bash
kubectl cluster-info
```

### 4.3. Master Taint'i Kaldırma

Single-node cluster'da, master node üzerinde pod'lar çalıştırabilmek için taint'i kaldırın:

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

## 5. Pod Network Eklentisinin Kurulumu (CNI)

### 5.1. Calico Network Eklentisini Kurma

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml
```

### 5.2. Kurulumun Tamamlanmasını Bekleme

```bash
kubectl get pods -n kube-system -w
```

Tüm pod'lar Running durumuna gelene kadar bekleyin (CTRL+C ile izlemeyi durdurabilirsiniz).

### 5.3. Node Durumunu Kontrol Etme

```bash
kubectl get nodes
```

Node'un "Ready" durumunda olduğunu doğrulayın.

## 6. Cluster'ın Test Edilmesi

### 6.1. Test Pod'u Oluşturma

```bash
kubectl run nginx-test --image=nginx
```

### 6.2. Pod'un Çalıştığını Kontrol Etme

```bash
kubectl get pods
```

### 6.3. Pod Detaylarını Görüntüleme

```bash
kubectl describe pod nginx-test
```

## 7. Kubernetes Dashboard Kurulumu (İsteğe Bağlı)

### 7.1. Dashboard Bileşenlerini Kurma

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

### 7.2. Dashboard için Admin Kullanıcısı Oluşturma

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
EOF

cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF
```

### 7.3. Dashboard için Token Oluşturma

```bash
kubectl -n kubernetes-dashboard create token admin-user
```

Çıktıda görünen token'ı kopyalayıp bir yere kaydedin.

### 7.4. Dashboard'a Erişim

```bash
kubectl proxy
```

Bu komutu çalıştırdıktan sonra, Dashboard'a şu URL ile erişebilirsiniz:
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

Token ile giriş yapmanız istenecektir, 7.3. adımda oluşturduğunuz token'ı kullanın.

## 8. Helm Kurulumu (İsteğe Bağlı)

### 8.1. Helm Yükleme

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### 8.2. Helm Versiyonunu Kontrol Etme

```bash
helm version
```

## 9. Sorun Giderme

### 9.1. Kubelet Durumunu Kontrol Etme

```bash
sudo systemctl status kubelet
```

### 9.2. Kubelet Loglarını İnceleme

```bash
sudo journalctl -u kubelet -f
```

### 9.3. Pod Durumunu İnceleme

```bash
kubectl get pods --all-namespaces
```

### 9.4. Pod Loglarını İnceleme

```bash
kubectl logs <pod-adı> -n <namespace>
```

### 9.5. Node Bilgilerini Görüntüleme

```bash
kubectl describe node k8s-node
```

## 10. Kullanışlı Kubernetes Komutları

### 10.1. Cluster Bilgilerini Görüntüleme

```bash
kubectl cluster-info
kubectl get componentstatuses
```

### 10.2. Namespace'leri Listeleme

```bash
kubectl get namespaces
```

### 10.3. Tüm Kaynakları Listeleme

```bash
kubectl get all --all-namespaces
```

### 10.4. Deployment Oluşturma

```bash
kubectl create deployment nginx-deployment --image=nginx --replicas=2
```

### 10.5. Service Oluşturma

```bash
kubectl expose deployment nginx-deployment --port=80 --type=NodePort
```

### 10.6. Service'i Görüntüleme

```bash
kubectl get services
```

### 10.7. Deployment'ı Ölçeklendirme

```bash
kubectl scale deployment nginx-deployment --replicas=3
```

## 11. Sistem Yedekleme ve Bakım

### 11.1. ETCD Verilerini Yedekleme

```bash
sudo apt install -y etcd-client
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save etcd-backup.db
```

### 11.2. Kubernetes Sürümünü Yükseltme

```bash
# Mevcut sürümü kontrol edin
kubectl version

# Kubeadm'ı yükseltin
sudo apt update
sudo apt-mark unhold kubeadm
sudo apt install -y kubeadm=X.Y.Z-00 (istediğiniz sürüm)
sudo apt-mark hold kubeadm

# Yükseltme planını kontrol edin
sudo kubeadm upgrade plan

# Yükseltmeyi uygulayın
sudo kubeadm upgrade apply vX.Y.Z

# Kubelet ve kubectl'i yükseltin
sudo apt-mark unhold kubelet kubectl
sudo apt install -y kubelet=X.Y.Z-00 kubectl=X.Y.Z-00
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

## 12. Güvenlik Önlemleri

### 12.1. API Server'a Erişimi Sınırlama

Kubernetes API Server'a, varsayılan olarak yerel ağ üzerinden erişilebilir. Güvenlik için API Server'a erişimi sınırlamak isteyebilirsiniz:

```bash
sudo nano /etc/kubernetes/manifests/kube-apiserver.yaml
```

`--allow-privileged=true` satırından sonra, aşağıdaki satırı ekleyin:

```
--enable-admission-plugins=NodeRestriction
```

Değişiklikten sonra kubelet'i yeniden başlatın:

```bash
sudo systemctl restart kubelet
```

### 12.2. Network Politikalarını Etkinleştirme

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF
```

## Sonuç

Bu dokümantasyon ile Ubuntu 22.04 üzerinde tek sunuculu (Single Node) Kubernetes kurulumunu başarıyla tamamlamış olmalısınız. Kubernetes ortamınız artık container uygulamalarını dağıtmaya ve yönetmeye hazır durumdadır.
