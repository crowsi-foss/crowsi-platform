# crowsi-platform
the crowsi platform is a kubernetes based orchastrator of honeypot decoys, purpose build to protect and surveil edge devices. As default it comes with the httpCatcherDecoy docker image deployed, which is a low interaction http api decoy.
It is available on docker hub and the code can be found here 
https://github.com/crowsi-foss/httpCatcherDecoy

Want to get more information on the crowsi-platform? Visit:


# deployment prerequisites
in order to set-up crowsi by yourself, you need:
- a running kubernetes cluster (currently the crowsi-platform has been tested with azure kubernetes service, but any other provider should work as well)
- helm installed on your administration set-up
- Traefik's helm chart repository, as crowsi makes use of the traefik proxy
- Cert-Managers helm chart repository, as crowsi uses cert-manager to retrieve and manages tls certificates from let's encrypt
- a mail address used to retrieve certificates from let's encrypt
- base64 string of the certificate of the CA that signed the certificates of your edge devices (see for more details). You can encode a certificate into base64 on a terminal by using this command "cat ca.crt | base64"


# deployment steps
1. Clone or download this repository
2. Install cert-manager on your kubernetes cluster (currently tested with version 1.13.2)
3. Install traefik on your kubernetes cluster, while adding a service annotation specifying the domain name under which your deployment endpoint shall be available. In Azure and traefik version 25.0.0. this can be done with the following command line. Please note that you need to chose a dns-label-name that is still available under azure, so best choose something random and check if deployment is executed successfully. 

helm install --values ./HelmTraefik/myvalues.yaml --set service.annotations."service\.beta\.kubernetes\.io/azure-dns-label-name"=someLabel traefik traefik/traefik --version 25.0.0


4. Install crowsi on your kubernetes cluster, while adding the base64 string of your CA certificate, the complete dns address of your deployment endpoint and your mail address.
helm install --set cacrt=base64string --set dnsName=DNSAddressOfYourEndpoint --set mail=YourMail crowsi ./HelmCrowsi/crowsi


# how to gain insights
Purpose of crowsi is to bind resources of attackers, interacting with valuable edge-device assets and create valuable insights by intensive logging. 
But therefore propper log monitoring is needed.
crowsi application logs are pushed directly to stdout and obviously you could just manually go through these logs. However, we recommend to integrate crowsi into your existing monitoring platfrom and then create alerts, dashboards and reports just as you need them.

Don't have a monitoring solution as of now?
There are many solutions out there, even some open source ones. To give one example, let's name the elastic stack and either the elastic agent or file beat as log collector. 


# Looking for a managed service and premium decoys?
operating and enhancing a honeypot solution can be though, therefore we are working on providing crowsi as a managed service platform.
Want to get more details? Go to 



