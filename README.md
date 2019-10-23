# Installation and demo of Red Hat Service Mesh on OpenShift

## Prerequisites
Admin access to a Red Hat Openshift cluster - or the single node version on your laptop *Code Ready Containers*

## Introduction - Openshift Service Mesh demo. 
OpenShift Service Mesh is based on the upstream Istio project, but with Openshift we add to it - particularly
in the areas of traffic visualisation - you'll see one such component in today's demo 
- Kiali - for service mesh topology visualisation

As you can see, I'm logged into my Openshift cluster as Administrator - both on the terminal and web interfaces.
We use K8S operators to install the service mesh. Provisioning of Operators requires Admin access
Consumption of operators - typically by developers does not. But for speed I'll use 
the same Admin user for both provisioning and use.

## Instructions
We need to install 4 Operators - from the Openshift Operator Hub. Navigate to the OpenShift Operator Hub:


The first 3 support the main one, the *Red Hat OpenShift Service Mesh Operator* 
- Elasticsearch
- Jaeger - for distributed tracing
- Kiali - for Service Mesh topology visualisation
- Red Hat OpenShift Service Mesh Operator

===
===
===

<first I create the Elasticsearch operator - at cluster scope - same for the rest >
<then I create the Jaeger Distributed Tracing operator >
<then I create the Kiali operator >
<Finally I create OpenShift Service Mesh Operator >

<pause recording>
After a few minutes later my operators have installed - as you can see on screen

Next I'm going to create a Service Mesh from the Service Mesh operator 
- earlier I created a project namespace <off the projects menu here - HOVER>  istio-system    
to hold my Service Mesh application. Let's select that

First I create a new Istio Service Mesh Control Plane in my namespace istio-system.
There are varios tunables here on screen - regarding the various components of SM
... I'll stick with the defaults
<pause recording>

After a short time later, my application is installed - and I can verify it on screen 
on my pods view here
<scroll through>
by checking the pods in my istio-system namespace - as follows:
clear
oc project istio-system
oc get pods -w


Now I'm ready to apply Service Mesh control to my microservices Application.

I'm going to use the Sample BookInfo application - available from the upstream Istio website.

here's a diagram of the application
< show diagram of the application >
It's a very basic application - a webpage called productpage. 
On the left hand side of the screen will be displayed the result of the details page.
Of most interest to us are the 3 reviews microservices.
- when v1 of reviews is called - ratings is not called and NO stars are shown
- when v2 of reviews is called - ratings are called and BLACK stars are shown
- when v3 of reviews is called - ratings are called and RED stars are shown

As we apply control - this visual representation makes it easy to see what's happening


3 things we need to do

<start recording with architecture diagram>


1) - the first step is to create namespace for the bookinfo application - let's call it bookinfo
I can do this either on the GUI or the command line - let's do it on the Command line
clear
oc new-project bookinfo

< ENSURE istio-system selected on GUI >
2) The next step is to create a 'Service Mesh Member Roll' - this essentially dictates which namespaces
I'm going to apply SM control to - so I'll just enter bookinfo

3) Finally I install my     bookinfo    microservices application - which my Service Mesh Member Roll
is looking out to apply control to


I'll do that by applying some yaml that installs the Bookinfo Microservices
application we just saw a diagram of
On seeing this creation - the Service Mesh Member Roll will apply Service Mesh control to it
clear
oc project bookinfo
oc apply -f https://raw.githubusercontent.com/istio/istio/release-1.3/samples/bookinfo/platform/kube/bookinfo.yaml

<pause recording>
# wait will it completes

==================================================================================================
A couple of minutes later - our Bookinfo Microservices application is installed - with service mesh control
as we can see <scroll through bookinfo pods>

Next we need to setup some Service Mesh constructs - inherited from Upstream Istio - for Service Mesh control
 - first the Envoy based side car proxies and the microservices to apply them to
clear
oc apply -n bookinfo -f https://raw.githubusercontent.com/Maistra/bookinfo/maistra-1.0/bookinfo.yaml

 - then an Istio gateway - representing the port and protocol at the ingress point to the mesh 
 - in our case HTTP and port 80
clear
oc apply -n bookinfo -f https://raw.githubusercontent.com/Maistra/bookinfo/maistra-1.0/bookinfo-gateway.yaml

 - next the Istio Destination rules - that is addressible services and their versions
clear
oc apply -n bookinfo -f https://raw.githubusercontent.com/istio/istio/release-1.1/samples/bookinfo/networking/destination-rule-all.yaml



export GATEWAY_URL=$(oc -n istio-system get route istio-ingressgateway -o jsonpath='{.spec.host}')
echo $GATEWAY_URL
We'll append the path /productpage to the gateway URL and hit it in a browser
This is the URL we'll hit throughout
<hit that in browser>

<pause recording>
<offline hit KIALI route to get past insecure warning - close window>

<show product page>
If we refresh this page several times we can see that it loops through the Review Microservice versions - showing
 - black stars
 - no stars
 - red stars
Now lets's move to Istio-system -> Networking -> Routes -> and open the Kiali route >
<put admin password     r3dh4t1!    on the clipboard>

Here we can see our topology visualisation tool - Kiali
It's going to show us the folow of traffic - in a very easy to visualise way
We'll use our OpenShift credentials to authenticate
<pause recording>

Now we're logged into Kiali - we'll choose
    namespace:      bookinfo
    left hand side: Graph
    Top
        versioned app 
        and moving traffic??????
        Requests %
        Select every 5 seconds - last 5 minutes

--------------------------------------------------------------------------------------------

# Now let's start to apply traffic control
# First - apply version1 of everything
We do this with virtual services - a virtual service defines a set of traffic routing rules 
to apply when a host is addressed - including the service version - in our case version 1 of all services
So let's apply this
clear
oc apply -f https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/networking/virtual-service-all-v1.yaml

Let's hit webpage maybe 10 times
We can see version 1  - with no stars is repeatedly being served

<pause recording till KIALI up to date>

If we switch to Kiali - we can see reviews - the only one with more than 1 version
is only being served by version 1

------------------------
# (jason - v2 (black); others v3(red))

Next, I'd like to demonstrate how Service Mesh can distinguish individual 
users in its routing decisions. This can be useful for Canary releases 
- where we only want to release a particular version to a particular group of users.
I'm going to login as user Jason - once logged in he will only see 
version 2 - with black stars - all others will see version 3 - with red stars

First let's apply that rule
clear
oc replace -f https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/networking/virtual-service-reviews-jason-v2-v3.yaml

    # login as jason/jason:     all v2 (black) 
    # Logout                    all v3 (red)

<refresh webpage repeatedly>
Now if we login as Jason - and refresh the page repeatedly - he only sees black stars
And if we logout - only red stars are shown
<refresh webpage repeatedly>

<pause recording till KIALI up to date>

after a few minutes if we hit Kiali - it shows only version 2 and 3 as expected.
------------------------
# 50% v2, 50% v3

Now let's say we want to retire version 1 and we're still not sure about whether we want to 
proceed with version 2 or 3. So we're going to send 50% to both - regeardless of who's logged in   
So we apply that rule
clear
oc replace -f https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/networking/virtual-service-reviews-v2-v3.yaml
<refresh webpage repeatedly>

and we see half with red and half with black stars as expected.
------------------------

# 100% v3
Now let's say we're happy with version 3 and we want to roll it out to 100% of traffic
Let's go ahead and do that
clear
oc replace -f https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/networking/virtual-service-reviews-v3.yaml

<pause recording till KIALI up to date>
and you can see Kiali reflects this

------------------------

Finally I'd like to demomstrate another feature of Service Mesh - Fault Injection
Netflix famously have a chaos monkey - that randomly just kills nodes and applications.
In service mesh we can do this also - to ensure your system recovers from faults
So we'll apply a fault injection - on our details microservice to always return an error
a HTTP status of 555
Let's apply this
clear
oc apply -f https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/networking/fault-injection-details-v1.yaml

<refresh webpage repeatedly>
We can see details on the left is always returning an error
<pause recording till KIALI up to date>

and we can see Kiali also representing this as an error.

So this has been a demo on installing and using OpenShift Service Mesh - based on the 
upstream Istio project.
We've only scratched the surface on what's possible - but hopefully you get a glimpse of
its powerful traffic control and visualisation capabilities



























## Prerequisites
Access to a 3scale Account  
Access to a Red Hat SSO installation  
An API client. We assume it to be Postman client for Chrome  
Your RHEL box IP, refered to *rhel-box-ip*. We assume RHSSO and 3scale on Prem are runnning on the same RHEL box with this IP. (If not substitute as appropriate below).  
  

Note: We have inserted whitespace in some URLs as they contain placeholders and are not valid URLs as they appear. Insert the values resolving to the URL fragments and remove whitespace.  

1 - Setup your 3scale Account
==================================================================================================
Go to API - Integration and click *edit integration settings*  
![Edit integration settings](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/01%20edit%20integration%20settings.png)
  
  
For GATEWAY, *choose APICast Self Managed*. For AUTHENTICATION choose *Oauth 2.0*. Then click Update Service  
![edit-integration-settings-details](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/01-2-edit-integration-settings-details.png)  
  
  
Click *add the base URL of your API and save the Configuration*  
![add-base-url](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/02%20Add%20the%20base%20URL%20of%20your%20API.png)  
Keep *Private Base URL* as it is (or enter your desired API Back end URL).  
Set your *Staging Public Base URL* and *Production Public Base URL* both to be http:// apicast-rshsso.*rhel-box-ip*.xip.io  
Click *Update the Staging Envirnonment*
![public-base-urls](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/03%20Set%20Public%20Base%20URLs.png)
Set your credentials location to Headers:  
![update-credentials-location-headers](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/03-update-credentials-location-headerz.png)
  
Go to API -> Integration. Click Promote v._x_ to Production
![promote-to-production](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/03-promote-toprodn.png)
  
Choose Developers - Account Developer - 1 Application  
![developer-application](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/04-open-developer-appl.png)

Open your only Application (or choose one in your Oauth API if you have more)
![set-application-details](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/04-set%20your-3scale%20app-details.png)
Add Random Key  
Set your Redirect URL to be https://www.getpostman.com/oauth2/callback
![application-details](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/05-get-your-3scale%20application-details.png)
Copy your API Credentials for later. Refered to as  
*client-id*  
*client-secret*  
*redirect-url*  
  
  
  
Next, get your 3scale Access token (used to create gateway). Go to Gear sign - Personal Settings - Tokens  
![05-threescale-accesstoken-1.png](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/05-threescale-accesstoken-1.png)
  
Add Access token  
![05-threescale-accesstoken-1.png](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/05-threescale-accesstoken-2.png)
  
Name it, give it read/write access to all scopes and Create Access token. Copy it (you won't be able to see it again).  
We'll refer to this your *3scale-access-token*   
![05-threescale-accesstoken-1.png](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/05-threescale-accesstoken-3.png)
  

2 - Initial Setup of Red Hat Single Sign On
==================================================================================================
Login to Master  
Create a Realm for 3scale Oauth. Name it. We'll refer to it as **_3scale-oauth-realm_** 
![6-add-realm-to-rh-sso.png](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/06-add-realm-to-rhsso.png)  
Test your realm hitting it in a browser  
http:// sso-rhsso.*rhel-box-ip*.xip.io/auth/realms/**_3scale-oauth-realm_**  (replacing **_3scale-oauth-realm_** with your realm name)  
  
  
Relax SSL only requirement (for a POC, never in production). As shown under the Login Tab - set it to none and Save.
![6-relax-ssl-requirement.png](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/07-relax-ssl-requirement.png)
  
  
Choose Initial Access Token (this is a token we will use for a Red Hat SSO API call shortly)  
![initial-access-token](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/08-create-initial-access-tkn.png)
  

Set your expiration and count to be non zero values. 
![9-set-expiration-and-count-999.png](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/09-set-expiration-and-count-999.png)  
Save it. Your Initial Access Token will appear. Copy it, you won't be able to retrieve it again. We'll refer to it as your *initial-access-token*
  
  
Create a user on RH SSO. For simplicity I'll refer to as *rh-sso-user-id*.
![10-Add-User.png](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/10-Add-User.png)  

Enter your equivalents to the following and Save:
![11-add-user-initial-details.png](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/11-add-user-initial-details.png)  

On the Credentials tab, add New Password and Password Confirmation and reset. I'll refer to it as rh-sso-user-password
![12-add-credentials.png](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/12-add-credentials.png)
  
  

3 - Use Postman to create client on Red Hat Single Sign On
==================================================================================================
In Postman, import the postman JSON *RHSSO_3scaleGist.postman_collection.json* in this repo.
Open RH SSO Add Client request add client to Red Hat SSO (Client credentials on RHSSO need to be the same as Application Credentials set on 3scale above)
Set the realm name in the address (to reflect *rhel-box-ip*) and the Bearer token to your *initial-access-token* retrieved above.
![12-postman-add-cli-1.png](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/13-postman-add-cli-1.png)
Save it. 
Switch to the Body tab. Enter the *client-id*, *client-secret* and *redirect-url* you set in 3scale above (Redirect URL is called redirectUris here)  
![14-Add-Client-Body.png](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/14-Add-Cli-Bdy.png)
  
Save and Send it. You should get back some JSON and 201 created HTTP status.  
  


4 - Check and Update your Client in Red Hat SSO  
==================================================================================================
In Red Hat SSO, click on the Clients view on the Left hand side. Select your new client (*client-id*)  
Assuming you want to give the user ability to grant access to the Application to access their data (we do), click Consent Required. Save  
![14-update-client-in-rhsso.png](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/14-update-client-in-rh-sso.png)  
  
  
5 - Setup your API Gateway on Openshift  
==================================================================================================
Login to Openshift and make the following commands
oc login 
	(default credentials are developer/developer)  
Create your project, e.g. with something like these values:  
oc new-project "3scalegateway-**_3scale-oauth-realm_**" --display-name="3scalegateway-**_3scale-oauth-realm_**" --description="3scalegateway-**_3scale-oauth-realm_**"   (replacing **_3scale-oauth-realm_** with your realm name) 
  
oc secret new-basicauth apicast-configuration-url-secret --password=https:// *3scale-access-token* @ **_3scale-domain_** (remove whitespace,  replacing **_3scale-domain_** with the URL to your SaaS on On Prem 3scale API Manager)    
  
oc new-app -f https://raw.githubusercontent.com/3scale/3scale-amp-openshift-templates/2.0.0.GA-redhat-2/apicast-gateway/apicast.yml  
  
Your gateway will deploy in a couple of minutes.  
**TIP** Scale it down to 1 Pod - that way if you want to look up logs, they can only be in the remaining Pod.  
  
Open web Console https:// *openshift-host* :8443 and open 3scalegateway-**_3scale-oauth-realm_**   (replacing **_3scale-oauth-realm_** with your realm name)  
Go to Applications -> Deployments -> apicast -> Environment  
Add this ENV variable: RHSSO_ENDPOINT and set it to: http:// sso-rhsso.*rhel-box-ip*.xip.io/auth/realms/**_3scale-oauth-realm_**   (replacing **_3scale-oauth-realm_** with your realm name)  
Save  
  
On Openshift web Console ->  Overview -> Create Route. 
Name: 		apicast-rshsso-route  
Hostname:	apicast-rshsso.*rhel-box-ip*.xip.io  
Leave the rest of the defaults and click Create  
![15-create-route.png](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/15-create-rout.png)  
  
  
6 - In Postman test Oauth flow and API with token  
==================================================================================================  
Open Echo Hello. Set your GET URL to be http:// apicast-rshsso.*rhel-box-ip*.xip.io/hello  
![16-echo-hello-request.png](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/16-echo-hello-requestt.png)  

Choose Type -> Oauth 2.0
![17-type-Oauth2.0.png](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/17-type-Oauth20.png)  
  
Set these parameters and  
Click Get New Access Token orange button  
Auth URL:			http:// apicast-rshsso.*rhel-box-ip*.xip.io/authorize  
Access Token URL:	http:// apicast-rshsso.*rhel-box-ip*.xip.io/oauth/token  
Client ID 			*client-id*  
Client Secret 		*client-secret*  
![18-Postman-Request-token.png](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/18-Postman-Request-tkn.png)  
  
Login as the user you created in RH SSO (rh-sso-user-id/rh-sso-user-password). (styling is always missing so ignore for now)  
![19-login-as-your-user.png](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/19-login-as-your-user.png)  
After authentication, you'll be prompted to create a new Password. Do this.  
  
A Token (RHSSO Get Token) will be returned to Postman. Click it then click Use Token.  
Click Save then click Send. You should get a successful API Response.    
![20-use-token-save-send.png](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/20-use-token-savesend.png)  

Go to your 3scale Analytics. Your counts should increment with each call.
![3scale-analytics](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/21-3scale-analytics.png)  
  
  
Return to postman and insert a random characted into the JWT and Use Token and repeat. Authorization should fail.
  
  

7 - Test your JWT on https://jwt.io (optional)  
==================================================================================================  
Go to https://jwt.io  
Paste into the Encoded box on the left the JWT that got inserted into the Bearer token Authorization Header after you selected Use Token.  
On the right, you'll see some decoded payload data representing the user and client.  
  
Go to Red Hat SSO -> Realm Settings -> Keys  
Copy the Public Key  
![21-get-public-key.png](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/21-get-public-key.png)
Insert Public Key as follows:  
-----BEGIN PUBLIC KEY-----
**_public key goes here_**
-----END PUBLIC KEY-----
			
Back in jwt.io, paste this into the Public Key or Certificate box on the right and the red Invalid Signature banner should turn to a blue *Signature verified* one.
Alter either the payload or the public key and signature validation will fail. This simulates the signature validation that hapeens on the gateway.
  
  
8 - Setup LDAP (if you don't have one, setup a test one using Open LDAP)
==================================================================================================
Go to: User Federation -> Add Provider -> ldap
![22-choose-ldap.png](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/22-choose-ldap.png)

Enter the following (or your equivalents), testing connection and authentication as you go then Save:
--------------------
Console Display Name:	open-ldap  
Priority				1  
Edit Mode				Read Only  
Vendor					Active Directory  
UUID LDAP attribute		entryUUID  
User Object Classes		shadowaccount, posixaccount  
Connection URL			ldap://*ldap-url*:389  
	***test it  
Users DN				ou=people,dc=example,dc=com  
Authentication Type		Simple  
Bind DN					cn=admin,dc=example,dc=com  
Bind Credential			*ldap-password*  
  
![22-sample-ldap-settings.png](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/22-sample-ldap-settingz.png)  
  
Retest OAuth flow and API with token In Postman - this time using credentials stored on LDAP  
  
  
  
9 - SSO into Dev Portal (based on https://support.3scale.net/docs/developer-portal/authentication#rhsso)
==================================================================================================
In 3scale, go to Settings -> Developer Portal -> SSO Integrations.  
![23-settings-dev-portal-sso.png](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/23-settings-dev-portal-sso.png)  
Click on Red Hat Single Sign On. Enter the following (or your desired values)  
**Client:**			3scale-dev-portal-client  
**Client secret:**	3scale-dev-portal-client-secret  
**Realm:**			http:// sso-rhsso.*rhel-box-ip*.xip.io/auth/realms/**_3scale-oauth-realm_**   (replacing **_3scale-oauth-realm_** with your realm name)  
![24-3scale-dev-portal-credentials.png](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/24-3scale-dev-portal-credentialz.png)  
  
  
While we're in 3scale we're going to remove the secret access key for easier access (for illustration only - we would normally keep it there until  our Dev Portal is ready to go live)  
In 3scale go to Settings -> Developer Portal. Delete the Developer Portal Access Code and Update Account.   
![25-remove-portal-access-key.png](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/25-remove-portal-access-keey.png)  
  
Next we need to add a client to your Realm in Red Hat SSO that aligns with your Dev Portal Oauth client to 3scale.  
Open Postman and repeat the steps in *3 - Use Postman to create client on Red Hat Single Sign On* above, replacing the data elements in the request JSON body with these (or your equivalents):  
**clientId:** 		3scale-dev-portal-client  
**secret:**			3scale-dev-portal-client-secret  
**redirectUris:**	https:// **_3scale-oauth-realm_**.3scale.net   (replacing **_3scale-oauth-realm_** with your realm name)  
(note the redirectUris entry is the same as your 3scale admin portal url without the '-admin')  
Click Send.  
  
You'll need to make a couple of mods to the client generated by your Postman call.  
Root URL:				https://**_3scale-oauth-realm_**.3scale.net 	(your dev portal URL, replacing **_3scale-oauth-realm_** with your realm name)  	
Valid Redirect URIs:	https://**_3scale-oauth-realm_**.3scale.net*	(your dev portal URL with * appended, replacing **_3scale-oauth-realm_** with your realm name)   
Web Origins:			**empty**							(delete what's there)  
![26-modify-3scale-portal-client-on-rhsso.png](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/26-modify-3scale-portal-client-on-rhsso.png)  
  
Test it out. Go to Developer Portal -> (in a new Incognito Window) Right click Visit Developer Portal - Open an Incognito Window 
![27-dev-portal-visit-dev-portal.png](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/27-dev-portal-visit-dev-portl.png)  
  
  
Click Sign In  
Authenticate with Red Hat SSO. Use either *rh-sso-ldap-user-id* or *rh-sso-user-id* 
![28-auth-dev-portal-with-rhsso](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/28-auth-dev-portal-with-rhsso.png)  

After logging in you'll be prompted to add Custom signup fields. Do this and follow the sequence to completion.
Close window  

Now as your 3scale Admin user, activate this Signup Request. Go to your Developers view and click Activate:  
  
![29-activate-signup-request](https://raw.githubusercontent.com/tnscorcoran/rhsso-3scale/master/29-activate-signup-request.png)    
  

Dev Portal signup is now complete and your Dev portal user can sign in with credentials stored in Red Hat SSO or an LDAP it brokers to.