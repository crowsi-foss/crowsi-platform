# crowsi-platform
the crowsi platform is a kubernetes based orchastrator of honeypot decoys, purpose build to protect and surveil edge devices. As default it comes with the httpCatcherDecoy docker image deployed, which is a low interaction http api decoy.
It is available on docker hub and the code can be found here 
https://github.com/crowsi-foss/httpCatcherDecoy

Want to get more information on the crowsi-platform? Visit:
www.crowsi.com

# deployment prerequisites
in order to set-up crowsi by yourself, you need:
- a running kubernetes cluster (currently the crowsi-platform has been tested with azure kubernetes service, but any other provider should work as well)
- helm installed on your administration set-up
- a mail address used to retrieve certificates from let's encrypt
- base64 string of the certificate of the CA that signed the certificates of your edge devices (see www.crowsi.com/documentation for more details). You can encode a certificate into base64 on a terminal by using this command "cat ca.crt | base64"

# dependencies
crowsi uses cert-manager and traefik to realize its fucntionality. Traefik is the proxy authenticating the edge-device client and exposing the decoys over a secure https connection and cert-manager is retrieving and managing the TLS certificates from let's encrypt.
In the following the necessary steps to install these will be described, but always checking the latest documenation of traefik and cert-manager is recomended.

crowsi is currently tested with
- cert-manager version 1.17.2
- traefik version 35.2.0




# deployment steps
1. Clone or download this repository and open a terminal inside the crowsi-platform folder
2. Install cert-manager on your kubernetes cluster. Run therefore the following code line

```
helm repo add cert-manager https://charts.jetstack.io
helm install cert-manager cert-manager/cert-manager --version 1.17.2 -f ./helm/cert-manager/certmanagervalues.yaml
```
   
3. Install traefik on your kubernetes cluster with the values stored in /helm/traefik/traefikvalues.yaml, while adding a service annotation specifying the domain name under which your deployment endpoint shall be available. In Azure and traefik this can be done with the following command line. Please note that you need to chose a dns-label-name that is still available under Azure, so best choose something random and check if deployment is executed successfully. 

```
helm repo add traefik https://traefik.github.io/charts
helm install --values ./helm/traefik/traefikvalues.yaml --set service.annotations."service\.beta\.kubernetes\.io/azure-dns-label-name"=someLabel traefik traefik/traefik --version 35.2.0
```


4. Install crowsi on your kubernetes cluster, while adding the base64 string of your CA certificate, the complete dns address of your deployment endpoint and your mail address.

```
helm install --set cacrt=base64string --set dnsName=DNSAddressOfYourEndpoint --set mail=YourMail crowsi ./helm/crowsi
```



# how to gain insights
Purpose of crowsi is to bind resources of attackers, interacting with valuable edge-device assets and create valuable insights by intensive logging. 
But therefore propper log monitoring is needed.
crowsi application logs are pushed directly to the kubernetes log space and obviously you could just manually go through these logs. However, we recommend to integrate crowsi into your existing monitoring platfrom and then create alerts, dashboards and reports just as you need them.

Don't have a monitoring solution as of now?
There are many solutions out there, even some open source ones. To give an example of a suitable platform, let's name the elastic stack and either the elastic agent or file beat as log collector. 


# Looking for a managed service and premium decoys?
operating and enhancing a honeypot solution can be tough, therefore we are working on providing crowsi as a managed service platform.
Want to get more details? Go to [www.crowsi.com/professional-services/](https://crowsi.com/professional-services/)




