## Cas 1 : Activer / désactiver un Admission Controller sur Minikube

Minikube permet de passer facilement des options à `kube-apiserver` via le flag `--extra-config`.  
Dans cet exemple, tu vas **activer le plugin `AlwaysPullImages`** et **désactiver `DefaultStorageClass`**.

### Étapes

1. **Supprimer le cluster existant si besoin :**
   ```bash
   minikube delete
   ```

2. **Créer un nouveau cluster avec les contrôleurs configurés :**
   ```bash
   minikube start \
     --extra-config=apiserver.enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,AlwaysPullImages,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota \
     --extra-config=apiserver.disable-admission-plugins=DefaultStorageClass
   ```

3. **Vérifier les Admission Controllers actifs :**
   ```bash
   minikube ssh
   sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep admission
   ```

   Tu devrais voir une ligne similaire à :
   ```
   --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,AlwaysPullImages,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota
   --disable-admission-plugins=DefaultStorageClass
   ```

4. **Tester le comportement :**
    - Crée un Pod avec une image déjà présente localement :
      ```bash
      kubectl run nginx --image=nginx
      ```
      Le contrôleur `AlwaysPullImages` forcera le téléchargement de l’image même si elle existe localement.

