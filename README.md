Signle-Sign-On with Azure Active Directory for HANA
===================================================

At last SAP Sapphire (May 2017) we announced several improvements and also new offerings for SAP on Azure as you can read [here](https://azure.microsoft.com/en-us/blog/the-best-public-cloud-for-sap-workloads-gets-more-powerful/). The most prominent ones are more HANA Certifications as well as SAP Cloud Platform on Azure (as you can read from [my last blog post specifically focused on SAP CP](http://blog.mszcool.com/index.php/2017/05/cloud-foundry-sap-cloud-platform-on-azure/).

One of the less discussed and visible announcements, despite being mentioned, is the broad support of Enterprise-Grade Single-Sign-On across many SAP technologies with [Azure Active Directory](https://azuremarketplace.microsoft.com/en-us/marketplace/apps/category/azure-active-directory-apps?page=1&search=sap). This post is solely about one of these offerings - [HANA integration with Azure AD](https://azuremarketplace.microsoft.com/en-us/marketplace/apps/aad.saphanadb?tab=Overview).

Pre-Requisites for HANA / AAD Single-Sign-On
--------------------------------------------
An integration of HANA with Azure AD (AAD) as primary Identity Provider works for HANA Instances you can run anywhere you want (on-premises, any public IaaS, [Azure VMs or SAP Large Instances in Azure](https://azure.microsoft.com/en-us/services/virtual-machines/sap-hana/)). The only requirement is, that the end-user accessing apps (Web Administration, XSA, Fiori) running inside of the HANA instance has access to the Internet to be able to sign-in against Azure AD.

For this post, I start with an SAP HANA Instance that runs inside of an Azure Virtual Machine. You can deploy such HANA instances [either manually](https://docs.microsoft.com/en-us/azure/virtual-machines/workloads/sap/hana-get-started#manual-installation-of-sap-hana) or through the [SAP Cloud Appliance Library](https://blogs.sap.com/2017/05/29/sap-cloud-appliance-library-now-supports-azure-resource-manager/).

In addition to just running HANA, I've also [installed XRDP for the Linux VM in Azure](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/use-remote-desktop) and [SAP HANA Studio](https://help.sap.com/viewer/a2a49126a5c546a9864aae22c05c3d0e/2.0.01/en-US) inside of the Virtual Machine to be able to perform necessary configurations across both, the XSA Administration Web Interface as well as HANA Studio as needd.

Finally, you need to have access to an Azure Active Directory tenant for which you are the Global Administrator or have the appropriate permissions to add configurations for [Enterprise Applications to that Azure AD Tenant](https://docs.microsoft.com/en-us/azure/active-directory/active-directory-enterprise-apps-manage-sso)!

The following figure gives an overview of the HANA VM environment I used for this blog-post. The important part is the [Azure Network Security Group](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-nsg) which opens up the ports for HTTP and HTTPS for HANA which are following the pattern 80xx and 43xx for regular HTTP and HTTPS, respectively.

![HANA VM in Azure Overview](https://raw.githubusercontent.com/mszcool/saphanasso/master/images/figure01.png)

Azure Active Directory Marketplace instead of manual configuration
------------------------------------------------------------------

SAP HANA is configured through the [Azure Active Directory Marketplace](https://azuremarketplace.microsoft.com/en-us/marketplace/apps/aad.saphanadb?tab=Overview) rather than the [regular App Registration model](https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-integrating-applications) followed for custom developed Apps in Azure AD. There are several reasons for this, here are the most important ones outlined:

* **SAML-P is required.**

  Most SAP assets follow SAML-P for Web-based Single-Sign-On. While it is possible in Azure AD, when setting it up manually with advanced options, Azure AD Premium Edition is required. For offerings from the Azure AD Marketplace (Gallery), standard edition is sufficient. While that's not the primary reason, it's a neat one!

* **Entity Identifier Formats for SAP Assets.**
  
  When registering an application in Azure AD through the regular App Registration model, Application IDs (Entity IDs in federation metadata documents) are required to be URNs with a protocol prefix (xyz://...). SAP applications use Entity IDs with arbirtray strings not following any specific format. Hence a regular app registration does not work. Again, this challenge can be solved through the Enterprise App Integration in AAD Premium. But when taking the pre-configured Offering from the Marketplace, you don't need to take care of such things!

* **Name ID formats in issued SAML Tokens.**

  Users are typically identified using Name ID assertions (claims). In requests, Azure AD accepts `nameid-format:persistent`, `nameid-format:emailAddress`, `nameid-format:unspecified` and `nameid-format:transient`. All of these are documented [here](https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-single-sign-on-protocol-reference) in detail. Now, the challenge here is:

  * HANA sends requests with `nameid-format:unspecified`.
  * This leads to Azure AD selecting the format for uniquely identifying a user.
  * But HANA expects the Name ID claim to contain the plain user name (johndoe instead of domain\johndoe or johndoe@domain.com).
  * This leads to a mismatch and HANA not detecting the user as a valid user even if the user exists inside of the HANA system!

  The Azure AD Marketplace item is configured and on-boarded in a way, that enables this technical challenge to be resolved.

* **Pre-configured claims**.

  While that's not a need for HANA in specific, for most of the other SAP-related offerings, the marketplace-based integrartion pre-configures the SSO-configuration with claims/assertions typically required by the respective SAP technology.

Step #1 - Register HANA in Azure Active Directory
-------------------------------------------------

Assuming you have HANA running in a VM as I explained earlier in this post, the first step to configure Azure AD as an Identity Provider for HANA is to add HANA as an [Enterprise Application to your Azure AD Tenant](https://docs.microsoft.com/en-us/azure/active-directory/active-directory-enterprise-apps-manage-sso). You need to select the offer as shown in the screen shot below:

![Selecting the HANA AAD Gallery Offering](https://raw.githubusercontent.com/mszcool/saphanasso/master/images/figure02.png)

Within the first step, you just need to specify a Display Name for the app as shown in the Azure AD management portal. The details are configured later down the road as next steps. Indeed you can get more detailed instructions from within the Azure AD management portal, directly. Just open up the `Signle Sign-On`-section, select `SAML-based Sign-On` in the very top Dropdown-Box, then scroll to the bottom and click the button for detailed demo instructions.

![Detailed Demo instructions for SAML-P](https://raw.githubusercontent.com/mszcool/saphanasso/master/images/figure03.png)

If you're filling out the SAML-P Sign-In settings according to these instructions, you're definitely on a good path. So, let's just walk through the settings so you get an example of what you need to enter there:

* **Identifier**: should be the Entity ID which HANA uses in it's Federation Metadata. Needs to be unique across all enterprise apps you have configured. I'll show you later down in this post, where you can find it. Essentially you need to navigate to HANA's Federation Metadata in the XSA Administration Web Interface.

* **Reply URL**: use the XSA SAML login endpoint of your HANA system for this setting. For my Azure VM, it had a public IP address bound to the Azure DNS name `marioszpsaphanaaaddemo.westeurope.cloudapp.azure.com`, therefore I had to configure `https://marioszpsaphanaaaddemo.westeurope.cloudapp.azure.com:4300/sap/hana/xs/saml/login.xscfunc` for it.

* **User Identifier**: this is one of the most important settings you must not forget. The default, `user.userprincipalname` will **NOT** work with HANA. You need to select a function called `ExtractMailPrefix()` in the Dropdown and select the `user.userprincipalname` for the `Mail`-parameter of this function.

![Detailed Settings Visualized](https://raw.githubusercontent.com/mszcool/saphanasso/master/images/figure04.png)

**Super-Important:** Don't ignore the *information*-message shown right below the certificate list and the link for getting the Federation Metadata. You need to check the box `Make new certificate active` so that the signatures will be correctly applied as part of the sign-in process. Otherwise, HANA won't be able to verify the signature.

Step #2 - Download the Federation Metadata from Azure AD
--------------------------------------------------------

After you have configured all settings, you need to save the SAML configuration before moving on. Once saved, you need to download the Federation Metadata for configuring SSO with Azure AD within the HANA administration interfaces. The previous screen-shot highlights the download-button in the lower, right corner.

Downloading the federation metadata document is the easiest way to get the required certificate and the name / entity identifier configured in your target HANA system.

Step #3 - Login to your HANA XSA Web Console and Configure a SAML IdP
---------------------------------------------------------------------

We have done all required configurations on the Azure AD side for now. As a next step, we need to enable SAML-P Authentication within HANA and configure Azure AD as a valid identity provider for your HANA System. For this purpose, open up the XSA web console of your HANA System by browsing to the respective HTTPS-endpoint. For my Azure VM, that was:

`https://marioszpsaphanaaaddemo.westeurope.cloudapp.azure.com:4300/sap/hana/xs/admin`

Of course, HANA will still redirect you to a Forms-based login-page because we have not configured SAML-P, yet. So, sign-in with your current XSA Administrator Account in the system to start the configuration.

**Tip:** take note of the Forms-Authentication URL. If you break something in your SAML-P configuration later down the road, you can always use it to sign back in via Forms Authentication to fix the configuration! The respective URL to take note of, is: `https://marioszpsaphanaaaddemo.westeurope.cloudapp.azure.com:4300/sap/hana/xs/formLogin/login.html?x-sap-origin-location=%2Fsap%2Fhana%2Fxs%2Fadmin%2F`.

Now the previously downloaded federation metadata document from Step #2 above becomes relevant. In the XSA Web Interface, you need to navigate to SAML Identity Providers and from there, click the "+"-button on the bottom of the screen. In the form opening now, just paste the previously downloaded federation metadata document into the large text box on the top of the screen. This will do most of the remaining job for you! But, you need to fix a few fields. 

* The name in the General Data must not contain any special characters, also no spaces.
* The SSO URL is not filled by default since we don't have it in the AAD metadata, yet. So you need to manually fill it as per the guidance from within the Azure AD portal shown above in this post.

![HANA SAML IdP Data Filled](https://raw.githubusercontent.com/mszcool/saphanasso/master/images/figure05.png)

Since we are in the HANA XSA tool, it's the right point in time to show you, where I retrieved the information required earlier in the Azure AD portal when registering HANA as an App there - the **Identifier** as shown in the last screen shot from the Azure AD console, above.

Indeed, these details are retrieved from the `SAML Service Provider` configuration section as highlighted in the screen shot below. A quick side-note: this is one of the rare cases where I constantly needed to switch to Microsoft Edge as a browser instead of Google Chrome. For some reasons, in Chrome I was unable to open the metadata tab, while in Edge I typically can open the metadata-tab which shows the entire Federation Metadata document for this HANA instance. From there, you can also grab the identifier required for Azure AD since this is the Entity ID inside of the Federation Metadata document.

![HANA SAML Federation Metadata](https://raw.githubusercontent.com/mszcool/saphanasso/master/images/figure06.png)

Ok, we have configured Azure AD as valid IdP for this HANA system. But we did not really enable SAML-based authentication for anything. This happens now at the level of applications managed by the XS-environment inside of HANA (that's how I understand this with my limited HANA knowledge:)). You can enable SAML-P on a per-package basis inside of XSA, means it's fully up to you to decide for which components you plan to enable SAML-P and for which you stay with other authentication methods. Below a screen shot that enables SAML-P for SAP-provided package! But stick with a warning: if you enable SAML-P for those, this might also have impact on other systems interacting with those packages. They should probably also support SAML-P as a means of authentication, especially if you disable other options, entirely!

![HANA SAML Federation Metadata](https://raw.githubusercontent.com/mszcool/saphanasso/master/images/figure07.png)

By enabling the `sap`-package for SAML-P, we get SSO based on Azure AD for a range of built-in functions including the XSA web interface, but also Fiori-interfaces hosted inside of the HANA instance for which you configured the setting.

Step #4 - Troubleshooting
-------------------------

So far so good, seems we could try it out, right? So, let's logout, open an `In-Private`-Browsing session with your browser of choice and navigate to your HANA XSA Administration application, again. You will see, that this time by default you will get redirected to Azure AD for signing into the HANA System. Let's see what happens when trying to login with a valid user from the Azure AD tenant.

![HANA SAML Federation Metadata](https://raw.githubusercontent.com/mszcool/saphanasso/master/images/figure08.png)

Seems the login was not so successful. Big question is why. This is now where we need access to the HANA system with HANA Studio and access to the system's trace log. For my configuraiton, I installed XRDP on the Linux machine and have the HANA Studio running directly on that machine. So, best way to start is connecting to the machine, starting HANA Studio and navigating to the system configuration settings.

![HANA Diagnosis for Sign-In Failing](https://raw.githubusercontent.com/mszcool/saphanasso/master/images/figure09.png)

The error-message is kind of confusing and miss-leading, though. We spent some time when onboarding HANA into the AAD Marketplace to figure out what was going wrong. So much ahead - Fiddler-Traces and issues with Certificates where not the problem! The resolution for this is to be found in an entirely different section. Nevertheless, I wanted to show this here, because it really is extremly valuable to understand, how-to troubleshoot stuff when it's not going well.

The main reason for this failure is a miss-match in timeout configurations. The signatures are created based on some time stamps. One of those timestamps is used for ensuring, that authentication messages are valid only a given amount of time. That time is set to a very low limit in HANA by default resulting into this, quite miss-leading error message.

Anyways, to fix it, you need to stay in the HANA System Level Properties within HANA Studio and make some tweaks and adjustments. Within the system properties of the Configuration Tab, just filter settings by SAML and adjust the `assertion_timeout` setting. It's impossible to do an entire, user-driven sign-in process within 10 sec. Think about it, the user navigates to a HANA App, gets re-directed to Azure AD, needs to enter her/his username/password, then eventually there's Multi-Factor-Auth involved and finally upon success, the user gets redirected back to the respective HANA application. Impossible within 10sec. So, in my case, I adjusted it to two min.

![HANA Diagnosis for Sign-In Failing](https://raw.githubusercontent.com/mszcool/saphanasso/master/images/figure10.png)

Ok, time for a next attempt. Now, if you still have the same error message about not being able to validate the signature, you probably forgot something earlier in the game. Make sure that when configuring HANA in Azure AD you make the certificate active by hitting the `Make new certificate active` checkbox I've mentioned earlier... below the same screen shot with the important informational message, again!

![Don't forget Make new certificate active](https://raw.githubusercontent.com/mszcool/saphanasso/master/images/figure04.png)

Step #4 - Configuring a HANA Database User
------------------------------------------

If you've followed all the steps so far, the Sign-In with a User from Azure AD will still not succeed. Again, the trace logs from HANA are giving more insights on what's going on and why the sign-in is failing this time.

![Trace about User does not exist in HANA](https://raw.githubusercontent.com/mszcool/saphanasso/master/images/figure11.png)

HANA is complaining that it does not know about the user. This is a fair complaint since Azure AD (or any other SAML Identity Provider) takes care for authentication, only. Authorization needs to happen in the actual target system (the service provider, also often called relying party application). For being able to Authorize, the user needs to be known to the service provdier. That means, at least some sort of user entity needs to be configured.

* With HANA, that means you essentially create a database user and enable Single Sign-On for this database user.

* HANA then uses the NameID-assertion from the resulting SAML token to assign the user authenticated by the IdP, Azure AD in this case, to map that successfully authenticated user to a HANA database user. This is where the format of the NameID in the issued token is so important and why we had to configure the `ExtractMailPrefix()`-strategy in the Azure AD portal as part of Step #1.

So, to make all of this happen and finally get to a successful login, we need to create a user in HANA, enable SSO and make sure that user has the appropriate permissions in HANA to e.g. access Fiori Apps or the XSA Administration Web Interface. This happens in HANA Studio, again.

![Detailed Settings Visualized](https://raw.githubusercontent.com/mszcool/saphanasso/master/images/figure12.png)

**Super-Important:** The left-most part of the figure above visualizes the mapping from the SAML-Token's perspective. So it defines the IdP as per the previous configurations and the user as it will end up being set in the NameID-assertion of the resulting SAML-token. With Azure AD users, these will mostly be lower-case! Case matters here!!! Make sure you enter the value lower-case here, otherwise you'll get a weird message about dynamic user creation failing!!!

The next step is to make sure that the user has the appropriate permissions. For me, as a non-HANA-expert, I just gave the user all permissions to make sure I can show success as part of this demo. Of course, that's not a best practice. You should give those permissions appropriate for your use cases, only.

![Detailed Settings Visualized](https://raw.githubusercontent.com/mszcool/saphanasso/master/images/figure13.png)

Step #5 - A Successful Login
----------------------------

Finally, we made it! If you have completed all the steps above, you can start using HANA with full Single-Sign-On across applications also integrated with your Azrue AD tenant. For example, the screen shot below shows my `globaladmin`-user account signing into the HANA Test VM I used, navigating to the HANA XSA Administration web console and then navigating from there to Office 365 Outlook... It all works like a charm without me being required to enter credentials, again!

![Detailed Settings Visualized](https://raw.githubusercontent.com/mszcool/saphanasso/master/images/figure14.png)

That is kind-of cool, isn't it! It would then even work with navigating back and forth between those environments. Now, this scenario would work for any application that runs inside of the XS-environment.

But for now, at least for enterprise administrators, it means they can secure very important parts of their HANA systems with a prooven Identity platform using Azure AD. They can even configure Multi-Factor Authentication in Azure AD and thus even further protect HANA environments along other applications using the same Azure AD tenant as an Identity Provider.

Final Words
-----------

Finally, this is the simplest possible way of integrating Single-Sign-On with SAP applications using Azure AD, only. SAP Netweaver would be similarly simple as it is documented [here](https://docs.microsoft.com/en-us/azure/active-directory/active-directory-saas-sap-netweaver-tutorial). There's even a more detailed tutorial available for Fiori Launch Pad on Netweaver based on these efforts we've implemented on [SAP blogs here](https://blogs.sap.com/2017/02/20/your-s4hana-environment-part-7-fiori-launchpad-saml-single-sing-on-with-azure-ad/).

The tip of the iceberg is then the most advanced SSO we've implemented with [SAP Cloud Platform Identity Authentication Services](https://docs.microsoft.com/en-us/azure/active-directory/active-directory-saas-sap-hana-cloud-platform-identity-authentication-tutorial). This will give you centralized SSO-management through both company's Identity-as-a-Service offerings (Azure AD, SAP Cloud Platform Identity Services). As part of that offering, SAP even includes automated identity provisioning which would remove the need for manually creating users as we did above.

I think, over the past year, we achieved a lot with the partnership between SAP and Microsoft. But if you ask for my personal opinion, I think the most significant achievements are HANA on Azure (of course, right:)), SAP Cloud Platform on Azure and ... the Single-Sign-On Offerings across all sorts of SAP technologies and services!

I hope you found this super-interesting. It is most probably my last blog post as member of the SAP Global Alliance Team from the technical side since I am moving forward to the customer facing part of Azure Engineering (Azure Customer Advisory Team) as an engineer. Still, I am part of the family and will engage as needed out of my new role with SAP, that's for sure!