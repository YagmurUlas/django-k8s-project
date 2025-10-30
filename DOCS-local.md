# Documentation

🔑 Step — Create a Django superuser (for /admin)
kubectl get pods

kubectl exec -it <django-pod-name> -- bash -c "python manage.py shell -c 'from django.contrib.auth import get_user_model; print(get_user_model().objects.filter(is_superuser=True))'"

----> Open the Node Port and Port Forward<----
kubectl edit svc -n ingress-nginx ingress-nginx-controller

find type: LoadBalancer and chane it to type: NodePort

Due to Kind limitations, you need to port-forward the Ingress:

```bash
sudo kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 80:80
```

Then access: http://django.local

You should see the Django homepage.
Admin panel is at:
👉 http://localhost:8080/admin

In production (AWS/GCP/Azure), LoadBalancer works automatically.

# To check ingress
kubectl get svc -n ingress-nginx


### Docker 
# Change to project directory
docker build -t your-dockerhub-username/django-app:v1 ./django-app
docker login
docker push your-dockerhub-username/django-app:v1

#### ARGOCD
update values.yml

create argocd manifest yaml file
deploy
 kubectl apply -f argocd-django-app.yaml
install argocd -> brew install argocd

argocd login localhost:8080 --username admin --password kS-eIJaYQ4D7PaJo --insecure


## ARGOCD sync

argocd app sync django-app



🧭 Notes and Troubleshooting

argocd app get django-app
