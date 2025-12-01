# K8s + Rocket.Chat + Traefik + cert-manager Setup Guide

This is the full walkthrough of how I set up Rocket.Chat on a K3s cluster with Traefik and Let's Encrypt SSL. I wrote this doc in a simple tone—clear, direct, and easy to follow.

---

## 1. Architecture Overview

**Components involved:**
- K3s (single-node cluster)
- Traefik (default ingress controller bundled with K3s)
- CoreDNS (cluster DNS)
- cert-manager (handles SSL issuing and auto-renewal)
- MongoDB (backend DB for Rocket.Chat)
- Rocket.Chat app (running as a Deployment)
- Ingress that exposes Rocket.Chat publicly with HTTPS

**Request Flow:**
User → Domain → Traefik LoadBalancer → Ingress → Rocket.Chat Service → Rocket.Chat Pod → MongoDB

---

## 2. DNS and Networking

### DNS Setup
Point your chosen domain to the server:
```
chat.your-domain.com     A     <your-server-public-ip>
```

### Firewall Rules Needed
- Allow port **80** (HTTP – required for Let's Encrypt challenge)
- Allow port **443** (HTTPS)
- Allow port **22** for SSH

Example (UFW):
```
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 22/tcp
```

---

## 3. Fixing Host DNS (Important)
The host must have working DNS so that pods can resolve external domains.

Create a clean `/etc/resolv.conf`:
```
sudo nano /etc/resolv.conf
```
Add:
```
nameserver 1.1.1.1
nameserver 8.8.8.8
options ndots:0
```

If systemd-resolved overwrites it:
```
sudo systemctl disable systemd-resolved --now
sudo rm /etc/resolv.conf
sudo nano /etc/resolv.conf
```

Restart networking and K3s:
```
sudo systemctl restart systemd-networkd
sudo systemctl restart k3s
```

---

## 4. Verifying Cluster DNS
Create a test pod:
```
kubectl run dns-test --image=busybox -- sleep 3600
```

Check DNS:
```
kubectl exec -it dns-test -- nslookup google.com
kubectl exec -it dns-test -- nslookup acme-v02.api.letsencrypt.org
```

If both resolve successfully → DNS is fixed.

Delete test pod:
```
kubectl delete pod dns-test
```

---

## 5. Installing cert-manager
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

Wait until all pods in the `cert-manager` namespace are ready.

---

## 6. Creating the ClusterIssuer (Let's Encrypt Production)
Create a file `cluster-issuer.yaml`:
```
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: rahul@redltd.tech
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: traefik
```
Apply:
```
kubectl apply -f cluster-issuer.yaml
```

Check status:
```
kubectl describe clusterissuer letsencrypt-prod
```
Should show `Ready=True`.

---

## 7. Deploying MongoDB & Rocket.Chat
Just the key environment variables required:

```
MONGO_URL=mongodb://mongo:27017/rocketchat?replicaSet=rs0
MONGO_OPLOG_URL=mongodb://mongo:27017/local?replicaSet=rs0
ROOT_URL=https://chat.your-domain.com       <replace with your domain>
PORT=3000
```

MongoDB must have a working replica set:
```
mongosh
rs.initiate({ _id: "rs0", members: [{ _id: 0, host: "mongo:27017" }] })
rs.status()
```

---

## 8. Creating the Rocket.Chat Ingress
Create `rocketchat-ingress.yml`:
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rocketchat-ingress
  annotations:
    kubernetes.io/ingress.class: traefik
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts:
        - chat.your-domain.com
      secretName: rocketchat-tls
  rules:
    - host: chat.your-domain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: rocketchat
                port:
                  number: 3000
```
Apply:
```
kubectl apply -f rocketchat-ingress.yml
```

Check:
```
kubectl get ingress rocketchat-ingress
```

---

## 9. Certificate Creation
Check certificate status:
```
kubectl get certificate
kubectl describe certificate rocketchat-tls
```

Check secret created:
```
kubectl get secret | grep rocketchat-tls
```

If the secret exists → SSL is active.
Open browser:
```
https://chat.red-cube.co.uk
```
Should show a valid HTTPS padlock.

---

## 10. SSL Auto-Renew
cert-manager will:
- monitor expiry
- renew 30 days before expiration
- update the secret
- Traefik will pick up the new cert
- No downtime

Check expiry date:
```
kubectl get certificate rocketchat-tls -o yaml | grep expiry
```

Manual renew (rarely needed):
```
kubectl delete secret rocketchat-tls
kubectl delete certificate rocketchat-tls
kubectl apply -f rocketchat-ingress.yml
```

---

## 11. Useful Debug Commands
### cert-manager logs
```
kubectl -n cert-manager logs deploy/cert-manager --tail=100
```

### Test HTTP challenge
```
curl -I http://chat.your-domain.com/.well-known/acme-challenge/test
```

### Check Traefik
```
kubectl -n kube-system get svc traefik -o wide
kubectl -n kube-system logs deploy/traefik
```

---

## 12. What “Healthy System” Looks Like
- `clusterissuer/letsencrypt-prod` → Ready=True
- `certificate/rocketchat-tls` → Ready=True
- `secret/rocketchat-tls` → exists
- Ingress shows both 80/443
- Browser shows valid SSL

At this point, the platform is production-ready.

## Here is our dashboard:

<img src="assets/rocketchat-ui-1.png" width="600">
<img src="assets/rocket-mobile-1.png" width="600">
<img src="assets/rocket-mobile-2.png" width="600">







---

If you ever want, I can export this doc as a PDF or create a GitHub-ready README.md version for your repos.

