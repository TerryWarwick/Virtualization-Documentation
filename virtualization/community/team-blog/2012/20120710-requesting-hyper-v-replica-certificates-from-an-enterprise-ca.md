---
title:      "Requesting Hyper-V Replica Certificates from an Enterprise CA"
description: Requesting Hyper-V Replica Certificates from an Enterprise CA
author: sethmanheim
ms.author: mabrigg
date:       2012-07-10 02:27:00
ms.date: 07/10/2012
categories: certificates
---
# Requesting Hyper-V Replica Certificates from an Enterprise CA
In an earlier [**post**](https://techcommunity.microsoft.com/t5/virtualization/requesting-hyper-v-replica-certificates-from-an-enterprise-ca/ba-p/381936), we discussed the steps required to get a certificate from a **Standalone CA** or from a third party CA. For an **Enterprise CA** , the INF file needs to be modified and suitable templates need to be available to honor the certificate request.

To make things interesting, the post is written for a deployment where both the primary and replica serves are part of a cluster where a **SAN** (Subject Alternative Name) **certificate** is being used for achieving certificate based authentication. It’s worth calling out two points:

  * The steps below can be used to create and manage wildcard and subject-name certificates by using an appropriate INF file (see earlier [post](https://techcommunity.microsoft.com/t5/virtualization/requesting-hyper-v-replica-certificates-from-an-enterprise-ca/ba-p/381936) for the INF file)

  * This post captures just one potential deployment model. The steps below will vary based on your enterprise CA policies and templates


**Step #1: Setup an Enterprise CA**

For this post, I have setup an Enterprise CA (called **frtest-new-ent-ca)** on Windows Server 2012 RC build. The configuration is straight forward and is achieved by enabling the Active Directory Certificate Services (ADCS) role and configuring an **Enterprise CA**.

To deploy SAN certificates, the CA needs to be configured to accept the SAN attribute from the request file. Issue the following commands from an elevated command prompt on the machine on which the Enterprise CA is configured:
    
```cmd
    certutil -setreg policy\EditFlags +EDITF_ATTRIBUTESUBJECTALTNAME2
    
    
    net stop certsvc
    
    
    net start certsvc
```
**Step #2: Manage/Create a template**

An out-of-box installation of the ADCS role does not have a template which can be reused for Hyper-V Replica. You would need to duplicate an existing template using the following steps:

  * Open ‘Certification Authority’ (certsrv.msc) from ‘Administrative Tools’
  * Right click on ‘ **Certificate Template'** and click on **Manage**



  * Right click on **Workstation Authentication** and choose the **Duplicate Template** option




  * The tabs which can be modified in the template are:

    * **Compatibility** : (Optional) Certificate recipient and authority can be set to Windows Server 8/Windows Server 2012

    * **General:** Provide display name, template name, validity period, renewal period as per your requirements.

    * **Security:** Ensure that **Authenticated Users** are allowed to Read and Enroll. Depending on your deployment, domain users need to have similar privileges but Authenticated Users should usually suffice. 

    * **Extensions:** Edit **Application Policies** and add **Server Authentication**

    * **Issuance Requirements:** (Optional) It is recommended that the CA certificate manager approves the request (as opposed to Auto enrollment)

    * **Subject Name:** Change the option to **Supply in the Request**




> Snips from a sample template (called **Hyper-VReplica** ) are shown below.
>
>
> **General:**
>
> **Compatibility:**  
>  
> **Extensions:**
>
> **Security:**  
>  
> **Issuance Requirements:**
>
> **Subject Name:**  
  



In case you missed the blog post introduction, this is just **one** **possible** template configuration which can be used to issue SAN certificates for Hyper-V Replica. If the above conditions are broadly met in other templates, skip step #2, and use any existing template.

Open “Certification Authority” on the server and click on “Certificate Templates”. Select “Action” (from top of the menu) and choose the “New” option followed by “Certificate Template to Issue”. Choose the certificate template name from the pop-up box (in this example “Hyper-VReplica” and click on the OK button.

**Step #3: Create SAN certificate on the primary cluster**

  * On any node in the primary cluster, retrieve the certificate for the certification authority by issuing the following command from an elevated command prompt


    
    
    certutil -f -config "servername\frtest-new-ent-ca" -ca.cert EntCA.CER

> Import this certificate into the **Trusted Root Certification Authorities** store of the Local computer. This is an optional step as the root certificate might already be installed in your servers.

  * Create an INF file which would allow you to request SAN certificate from the Enterprise CA


    
    
    [Version] 
    
    
    Signature="$Windows NT$ 
    
    
     
    
    
    [NewRequest]
    
    
    Subject = "CN=SANCert" ; Can be any server name/string
    
    
    Exportable = TRUE; Private key is exportable
    
    
    KeyLength = 2048; Common key sizes: 512, 1024, 2048, 
    
    
    KeySpec = 1             
    
    
    KeyUsage = 0xA0     ; Digital Signature, Key Encipherment
    
    
    MachineKeySet = True
    
    
    ProviderName = "Microsoft RSA SChannel Cryptographic Provider"
    
    
    ProviderType = 12
    
    
    RequestType = CMC
    
    
        
    
    
    [RequestAttributes]
    
    
    CertificateTemplate = "Hyper-VReplica" ; Name of the template which was created
    
    
    SAN="dns=demo-pri1.contoso.com&dns=demo-pri2.contoso.com&dns=pribroker.FRTEST.nttest.microsoft.com"  ; FQDN of the nodes which make up the cluster, plus the Hyper-V Replica Broker CAP

> Save the above file as **SAN.inf**

  * Create a new request from the inf file


    
    
    certreq -new SAN.inf SAN.req

  * Submit the request to the Enterprise CA


    
    
    certreq -submit -config "servername\frtest-new-ent-ca" SAN.req SAN.cer

  * Work with your CA admin to issue the pending request. In this example, the request ID is **19**


  * Once the request is issued, on the primary node, issue the following command to retrieve the response (cer file) from the CA. The request ID is 19 in this example.


    
    
    certreq -retrieve 19 SAN.cer

  * Import this certificate into the Personal store of the Local machine. Once imported, the entry would look as follows (the **Issued to** attribute is set to the Subject name specified in the INF file)


  * Export the SANCert (with the private key) as a pfx file
  * Import the pfx file on each node of the cluster in the **Personal** store of the Local Machine. Ensure that the root certificate is available in each node as well.



**Step #4: Create SAN certificate on the replica cluster**

  * Repeat the same steps as Step #3 after modifying the dns entries (under **RequestAttributes** ) in the inf file



That’s it, you are good to go! You now have a setup which has SAN certificates which have been issued by your Enterprise CA. These certificates can be used on the replica and primary cluster to receive and enable replication respectively.
