NETBOX STAGE:

	# Liberar acesso ao superuser no projeto:
	$ oc adm policy add-scc-to-user anyuid -z default -n netbox-stage

Criar os seguintes PVCs:
	- netbox-static-files (1GB - RWO)
	- netbox-nginx-config (1GB - RWO)

Criar os configmap:
	    $ oc create configmap nginx-conf --from-file=~/netbox-docker/docker/nginx.conf

1. PostgreSQL:
	$ oc new-app postgresql
	$ oc set env --from=secret/postgresql-secret dc/postgresql
	$ oc volume dc/postgresql --add -t pvc --claim-name=postgresql-data --name=postgresql-volume-1 --overwrite 

2. NETBOX:
	$ oc new-app --name=netbox ninech/netbox:latest-ldap
	$ oc volume dc/netbox --add --name=netbox-volume-2 -m /opt/netbox/netbox/static --overwrite
	$ oc volume dc/netbox --add --name=netbox-volume-2 --claim-name=netbox-static-files -t pvc --overwrite
	$ oc volume dc/netbox --add --name=netbox-volume-1 --claim-name=netbox-nginx-config -t pvc --overwrite

    # Inserir envs do 'netbox.env' no deploymentconfig yml

    # Criar SVC/netbox e expor porta 8001
	$ oc create -f netbox-svc.yml
	OU
	***$ oc expose svc netbox --port=8001

3. NGINX:
    	$ oc new-app nginx:1.13-alpine
	$ oc volume dc/nginx --add --name=nginx-volume-2 --claim-name=netbox-static-files -t pvc -m /opt/netbox/netbox/static --overwrite
	$ oc volume dc/nginx --add --name=nginx-volume-1 --claim-name=netbox-nginx-config -t pvc -m /etc/netbox-nginx/ --overwrite
	
   # Adicionar porta 8080 ao SVC/NGINX:
	***$ oc expose svc nginx --port=8080
      OU
	$ oc edit service nginx
	  - name: 8080-tcp
	    port: 8080
	    protocol: TCP
	    targetPort: 8080

   # Mapear configmap nginx-conf
	
          volumeMounts:
            - mountPath: /etc/nginx/nginx.conf
              name: nginx-conf
              subPath: nginx.conf
            - mountPath: /etc/netbox-nginx/nginx.conf
              name: nginx-netbox
              subPath: nginx.conf
      volumes:
        - configMap:
            defaultMode: 420
            name: nginx-conf
          name: nginx-conf
        - configMap:
            defaultMode: 420
            name: nginx-conf
          name: nginx-netbox
