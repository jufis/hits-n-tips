#Running Maximo Asset Management in WebSphere Portal using IFRAME in SSO environment

##A. SCOPE
Run maximo on websphere portal using an iframe or web doc app. This mut be using SSO ie. login to portal will auto login to maximo as well. Solution must not leak maximo ui sessions otherwise maximo will run out of resources or result in a DOS situation where no more users are allowed to login.

##B. GENERAL REFFERENCES
http://www-01.ibm.com/support/docview.wss?uid=swg21625523

http://www-01.ibm.com/support/docview.wss?uid=swg21633574

http://www-01.ibm.com/support/docview.wss?uid=swg21569050

https://www.ibm.com/developerworks/community/blogs/a9ba1efe-b731-4317-9724-a181d6155e3a/entry/maximo_and_ldap_configuration_from_start_to_finish?lang=en

##C. REQUIREMENTS
I assume that the following are already in place:

1. VM with TDS 6.3 already installed
2. VM with Maximo 7.5 already installed and working with IHS
3. VM with WebSphere Portal 8.5 already installed and working without IHS
4. All VMs must reffer to the other under the domain rural.gr so all host files need to be updated accordingly or run a dns service
5. All VMs must run under NTP server otherwise SSO will be unsuable.

##D. TECHNICAL JOURNEY

Get a deep breath first and try to follow me :)

##0. TDS TREE SETUP

Disabled sha256 to allow plain text passwds
That allows you to use Apache Directory Studio without a problem if your TDS is setup to use SHA256.

Refferences
https://www.ibm.com/developerworks/community/wikis/home?lang=en#!/wiki/Anything%20about%20Tivoli/page/Enable%20SHA-256%20as%20password%20encryption%20algorithm%20in%20Tivoli%20Directory%20Server%206.3
https://www-01.ibm.com/support/knowledgecenter/ssw_ibm_i_61/rzahy/rzahypwdencrypt.htm

Actions

    {TDS INSTALLED DIRECTORY}\bin\idsldapmodify -p 389 -D cn=root -w password -i addvar.ldif
    The "addvar.ldif" file contains the following:
    dn: cn=Configuration
    changetype: modify
    replace: ibm-slapdUseNonFIPSCrypt
    ibm-slapdUseNonFIPSCrypt: false


Add a new admin user to TDS

    Use idsldapadd to add the following user:


    C:\Program Files\IBM\LDAP\V6.3\bin>idsldapadd.cmd -p 1389 -D "cn=root" -w "Obj5ct00"
    dn: cn=jufis, cn=AdminGroup, cn=Configuration
    cn: jufis
    ibm-slapdAdminDN: cn=jufis
    ibm-slapdAdminPW: jufis13
    ibm-slapdAdminRole: AuditAdmin
    ibm-slapdAdminRole: DirDataAdmin
    ibm-slapdAdminRole: SchemaAdmin
    ibm-slapdAdminRole: ServerStartStopAdmin
    objectclass: top
    objectclass: ibm-slapdConfigEntry
    objectclass: ibm-slapdAdminGroupMember

For our case I'm attaching the whole tree in an ldif so you can import to your ldap.

##1. WAS REPOSITORY FOR TDS SETUP

Login to WAS dmgr console and goto Security->Global Security->Configure.
Create a new TDS repo and configure federation as follows:

    Realm name
    LDAP
    Primary admin user
    wasadmin
    Ignore case for authorization
    [selected]
    Base DN
    o=rural,C=GR
    Login properties in TDS
    uid;cn
    Support referrals to other LDAP servers 
    [ignored]
    Bind user
    cn=wasadmin,ou=users,o=rural,C=GR
    Ldap entity types
    Group	groupOfNames  (search filter 1=2)
    OrgContainer	organization;organizationalUnit;domain;container  
    PersonAccount	inetOrgPerson  
    Group attribute definition
    scope->direct
    member attributes: member (name), direct (scope), groupOfNames (object class)
    dynamic member attributes: none
    Federated repos->supported entity types
    Group	           ou=groups,o=rural,C=GR	      cn  
    OrgContainer	    ou=users,o=rural,C=GR	      o;ou;dc;cn  
    PersonAccount       ou=users,o=rural,C=GR             uid;userPrincipalName  
    Federated repositories > Trusted authentication realms - inbound
    [selected] 

    Trust realms as indicated below
    LDAP (name) 	Trusted (inbound trust)


##2. DISABLE INTERNAL FILE REPOSITORY NOT REQUIRED

    Goto global sec->configure. Select and delete file repository.


##3. Quirks

    Restart dmgr
    Check dmgr wasadmin login
    Cleanup temp, wstemp, logs, translog of dmgr and mxserver profiles manually by first stopping the servers.
    Manual sync nodeageant of maximo
    Start dmgr, nodeagent and check that they sync
    Stop both dmgr and node agent


##4. SETUP Maximo DB for app security

    # su - ctginst1
    # db2 connect to maxdb75
    # db2 "update maximo.maxpropvalue set propvalue='1' where propname='mxe.useAppServerSecurity'"


##5. Enable security in web.xml in maximo product ears

    a. cd /opt/IBM/WebSphere/AppServer/profiles/ctgAppSrv01
    b. cd installedApps
    c. find . -name web.xml
    d. for each web.xml vi it and remove comments between <security-constraint/> tags
    after the sec constrains you will find an env property useAppServerSecurity set to 0, turn this to 1
    e. do the same (a-d) for the profile of dmgr


##6. START MAXIMO UP

    a. start dmgr
    b. start nodeagean
    c. start mxserver
    d. start ihs
    e. check you can login in maximo with ldap now


##7. MAXIMO VMMSYNC CONFIG

System Configuration->Platform Configuration->Crontask setup->VMM SYNC

(Running as user MAXADMIN every 5 mintues)

    ChangePolling
    0 


    GroupMapping
    <?xml version="1.0" encoding="UTF-8" ?><!DOCTYPE ldapsync SYSTEM "ldapgroup.dtd"><ldapsync><group><basedn>ou=maxusers,o=rural,C=GR</basedn> <filter>Group</filter>  <scope>subtree</scope> <attributes> <attribute>cn</attribute> <attribute>description</attribute> </attributes> <datamap> <table name="MAXGROUP"> <keycolumn name="GROUPNAME" type="ALN">cn</keycolumn> <column name="DESCRIPTION" type="ALN">description</column>            </table></datamap> <memberdatamap> <membertable name="GROUPUSER"> <keycolumn name="GROUPNAME" type="ALN">cn</keycolumn> <membercolumn name="USERID" type="UPPER"> <member>member</member> <memberuser>uid</memberuser> <membergroup>cn</membergroup> </membercolumn> <column name="GROUPUSERID" type="INTEGER">{:uniqueid}</column> </membertable> </memberdatamap> </group></ldapsync> 


    GroupSearchAttribute
    cn 


    Principal
    cn=wasadmin,ou=users,o=rural,C=GR 


    Credential
    the passwd of the principal (maximo2013@) 


    Realm
    nothing is defined in here 


    UserMapping
    <?xml version="1.0" encoding="UTF-8" ?>
    <!DOCTYPE ldapsync SYSTEM "ldapuser.dtd">
    <ldapsync>
    	<user>
    		<basedn>ou=users,o=rural,C=GR</basedn>
    		<filter>PersonAccount</filter>
    		<scope>subtree</scope>
    		<attributes>
    			<attribute>uid</attribute>
    			<attribute>givenName</attribute>
    			<attribute>sn</attribute>
    			<attribute>displayName</attribute>
    			<attribute>street</attribute>
    			<attribute>telephoneNumber</attribute>
    			<attribute>mail</attribute>
    			<attribute>st</attribute>
    			<attribute>postalCode</attribute>
    			<attribute>c</attribute>
    			<attribute>l</attribute>
    		</attributes>
    		<datamap>
    			<table name="MAXUSER">
    				<keycolumn name="USERID" type="UPPER">uid</keycolumn>
    				<column name="LOGINID" type="ALN">uid</column>
    				<column name="PERSONID" type="UPPER">uid</column>
    				<column name="TYPE" type="UPPER">{TYPE 3}</column>
    			</table>
    			<table name="PERSON">
    				<keycolumn name="PERSONID" type="UPPER">uid</keycolumn>
    				<column name="STATUSDATE" type="ALN">{:sysdate}</column>
    				<column name="FIRSTNAME" type="ALN">givenName</column>
    				<column name="LASTNAME" type="ALN">sn</column>
    				<column name="DISPLAYNAME" type="ALN">displayName</column>
    			</table>
    		</datamap>
    	</user>
    </ldapsync> 


    UserSearchAttribute
    uid


##8. ENABLE VMMSYNC LOGGING

    System Configuration->Platform Configuration->Logging
    Click on crontask
    Bottom pane press new to add VMMSYNC logger and set to debug


##9. PROOF THAT VMM SYNC WORKS

    [10/7/15 12:08:14:092 EEST] 0000006b SystemOut     O 07 Oct 2015 12:08:14:092 [DEBUG] [MXServer] [CID-CRON-394] Synchronizing VMM Users: VMM search results 
    <?xml version="1.0" encoding="UTF-8"?>
    <sdo:datagraph xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xmlns:sdo="commonj.sdo" xmlns:wim="http://www.ibm.com/websphere/wim">
      <wim:Root>
        <wim:entities xsi:type="wim:PersonAccount">
          <wim:identifier externalName="cn=jufis,ou=users,o=rural,C=GR" repositoryId="tdsldap"
              uniqueId="CN=JUFIS,OU=USERS,O=RURAL,C=GR" uniqueName="cn=jufis,ou=users,o=rural,C=GR"/>
          <wim:uid>jufis</wim:uid>
          <wim:sn>kossionis</wim:sn>
          <wim:displayName>programming motherfucker</wim:displayName>
          <wim:givenName>georgios</wim:givenName>
        </wim:entities>
        <wim:entities xsi:type="wim:PersonAccount">
          <wim:identifier externalName="cn=wpadmin,ou=users,o=rural,C=GR" repositoryId="tdsldap"
              uniqueId="CN=WPADMIN,OU=USERS,O=RURAL,C=GR" uniqueName="cn=wpadmin,ou=users,o=rural,C=GR"/>
          <wim:uid>wpadmin</wim:uid>
          <wim:sn>wpadmin</wim:sn>
          <wim:displayName>wpadmin</wim:displayName>
          <wim:givenName>wpadmin</wim:givenName>
        </wim:entities>
        <wim:entities xsi:type="wim:PersonAccount">
          <wim:identifier externalName="cn=wasadmin,ou=users,o=rural,C=GR" repositoryId="tdsldap"
              uniqueId="CN=WASADMIN,OU=USERS,O=RURAL,C=GR" uniqueName="cn=wasadmin,ou=users,o=rural,C=GR"/>
          <wim:uid>wasadmin</wim:uid>
          <wim:sn>wasadmin</wim:sn>
          <wim:displayName>wasadmin</wim:displayName>
          <wim:givenName>wasadmin</wim:givenName>
        </wim:entities>
        <wim:entities xsi:type="wim:PersonAccount">
          <wim:identifier externalName="cn=mxintadm,ou=users,o=rural,C=GR" repositoryId="tdsldap"
              uniqueId="CN=MXINTADM,OU=USERS,O=RURAL,C=GR" uniqueName="cn=mxintadm,ou=users,o=rural,C=GR"/>
          <wim:uid>maxintadm</wim:uid>
          <wim:sn>integratoras</wim:sn>
          <wim:displayName>max integrator</wim:displayName>
          <wim:givenName>max</wim:givenName>
        </wim:entities>
        <wim:entities xsi:type="wim:PersonAccount">
          <wim:identifier externalName="cn=maxreg,ou=users,o=rural,C=GR" repositoryId="tdsldap"
              uniqueId="CN=MAXREG,OU=USERS,O=RURAL,C=GR" uniqueName="cn=maxreg,ou=users,o=rural,C=GR"/>
          <wim:uid>maxreg</wim:uid>
          <wim:sn>regas</wim:sn>
          <wim:displayName>max regas</wim:displayName>
          <wim:givenName>max</wim:givenName>
        </wim:entities>
        <wim:entities xsi:type="wim:PersonAccount">
          <wim:identifier externalName="cn=maxadmin,ou=users,o=rural,C=GR" repositoryId="tdsldap"
              uniqueId="CN=MAXADMIN,OU=USERS,O=RURAL,C=GR" uniqueName="cn=maxadmin,ou=users,o=rural,C=GR"/>
          <wim:uid>maxadmin</wim:uid>
          <wim:sn>admin</wim:sn>
          <wim:displayName>god</wim:displayName>
          <wim:givenName>max</wim:givenName>
        </wim:entities>
      </wim:Root>
    </sdo:datagraph>


##10. MAXIMO SETUP SSO and others

Choose a domain

    In our case our fictionary domain is rural.gr. Both servers maximo and portal will lie in rural.gr domain.


Export LTPA Keys

    Security->Global Security->LTPA
    Click generate keys
    Set passwd and a path and export ltpa keys to /tmp/ltpa.keys


Cookie setup

    Goto: Application servers > MXServer > Web container > Session management > Cookies
    In Cookie Domain set: rural.gr
    [make sure your /etc/hosts has an entry now for maximo.rural.gr in maximo vm ip]
    Change the JSESSIONID to MAXJESSIONID to avoid strange log outs from portal.


SSO setup

    Goto: Global security > Single sign-on (SSO)
    Select Enabled
    Domain name: rural.gr
    Select Interoperability Mode  
    Select Web inbound security attribute propagation  
    Leave require ssl off


Other Servers setup (ie. portal)

    You need to repeat the SSO setup with other WAS servers (ie. portal) but instead of exporting the ltpa keys you will import them with the passwd that you exported'em before. When you import ltpa keys you must restart dmgr, manual sync nodes and restart any other servers in your infra.


##11. PORTAL LDAP

Follow chaotic steps in pdf attached.

##12. Portal IFRAME Test

Let's try the portal home page now with an iframe in a wcm article pointing to maximo:

Inline image 1

That's good because we haven't login to portal yet! (Yes we can change the login portal form to something else or choose to not display the iframe when in public mode ie. not logged in)

Let's try to login to portal now and see what it will bring up:

Inline image 2

Looks good, in my iframe code I point to the SR application so this is what we see in here!

Now let's see how is logged in to maximo (always via portal):

Inline image 3

Apparently that's us! Maximo is configured to allow only 1 uisession per http session. Let's browse a little between apps and see if that causes any problems and then come back to user sessions and see what's in there. Problems found as this configuration currently leaks uisessions and during portal page browsing every time we go back to portal we general a new logged-in user for maximo. That is not good because it will leak sessions and exaust server resources.

##13. ISSUES with maximo uisessions

A. Changing portal pages iframe navigation is reset
Meaning that is you are editing the sr and goto another portal page or event think of reloading the portal page it will drive iframe navigation to the beginning ie. the start center for example.

B. Maximo redirects
Maximo redirects for various reasons such as not enough sessions, on 404 errors etc. If that happens the portal disappears and instead all you get is maximo.

C. Weird logouts
Portal was logging out for no reason while experimenting with maximo. This is persisting and I guess this is only happening because we have 2 jsessionid of 2 different servers fighting with each other I'll change the name of maximo jsessionid to maxjsessionid and see if that makes any difference. (This was tried out and works, so renaming the maximo jsessionid resolves this problem). Running servers under NTP will make this more stable.

D. Maximo UI Session ID exaustion
When you navigate from the iframe to other portal pages and back maximo creates a new uisession every time we follow this url:

    https://maximo.rural.gr/maximo/ui/maximo.jsp?event=loadapp&amp;value=sr&amp;additionalevent=useqbe&amp;portalmode=true


According to the following refferences maximo enforces different uisession per browser tab:

http://www-01.ibm.com/support/docview.wss?uid=swg21621635
https://www.ibm.com/developerworks/community/blogs/a9ba1efe-b731-4317-9724-a181d6155e3a/entry/understanding_concurrent_browser_support_in_tpae1?lang=en

I have altered the following properties in maximo in order to try to avoid this effect:

    mxe.webclient.allowURLDefinedUISessionID=0
    mxe.webclient.maxUISessionsPerHttpSession=1
    mxe.webclient.maxuisessions=0 (no chances here causes infinite sessions)
    mxe.webclient.maxuisessionsend503=0


This results in no effect. There is no way to remove the uisessionid or disable it. Or use it in another way (ie. using a cookie instead of a url param).

Possible Hack? Yes, we could try after iframe is loaded to set a cookie with js and store the iframe url, then add another hook and if there is a value in there to replace the iframe src to last known location; this way the user will not have to renavigate when changing portal pages. (see 14)

Possible Hack? Maybe using a custom java filter in maximo. Needs further investigation to not break the whole maximo.

Alternative way? Yes, portal web bridge. We need to try this in case it behaves better (see 13).

Alternative way? Yes, just use a button to popup a maximo screen that the IHS/EDGE of the edge infra reverse proxy using the same domain as portal is lying on but on a different path like this: http://portal.rural.gr/maximo

##14. Portal App Web Bridge Test

This is a reverse proxy thing that clips web server content we need to use this and see if this is making it any better/harder or worse. After testing here we concluded that this is not working for maximo at all. It brings the content but you cannot navigate around maximo as that causes numerous js errors.

##15. Testing of IFRAME with cookie hack and maximo reverse proxy from portal avoiding XSS

NOTE: disable compression between maximo ihs and apache24

Here we test iframe with maximo served as a reverse proxy from IHS of portal to avoid XSS story so as allow the javascript code to drive iframe the way we like. What we like to do here is to store the last known navigation point (url) and restore this from a cookie when the portal is restoring (or going back) to the portal page that maximo iframe lies.

First we installed ihs and setup admin server on windows:

    c:/ibm/httpserver/bin/htpasswd –b admin.passwd wpadmin wpadmin
    move admin.passwd ../conf


Started admin server:

    cd c:/ibm/httpserver/
    bin/httpd -f conf/admin.conf


Then configured for virtual hosts in ibm console and then registered apache as a win service:

    cd c:/ibm/httpdserver/bin

httpd -k install -n IBMHTTPServerV8.5 -f c:\IBM\HTTPServer\conf\httpd.conf 

NOTE: the -f param didn't got in place for some reason and I had to went to the win service and put it in there myself.

Registered admin server as a win service as well:

httpd -k install -n IBMAdminHTTPServerV8.5 -f c:\IBM\HTTPServer\conf\admin.conf 

NOTE: the -f param didn't got in place for some reason and I had to went to the win service and put it in there myself.

Configured was plugin:

    C:\IBM\WebSphere\Plugins\bin>ConfigureIHSPlugin.bat -plugin.home c:\IBM\WebSpher
    e\Plugins -plugin.config.xml c:\IBM\WebSphere\Plugins\config\webserver1\plugin-c
    fg.xml -ihs.conf c:\IBM\HTTPServer\conf\httpd.conf -WAS.webserver.name webserver
    1 -operating.system.arch 32


Configured certificate trust between plugin and was.

Configured the reverse proxy part of maximo into portal ihs httpd.conf and restarted ihs:

    LoadModule proxy_module modules/mod_proxy.so
    LoadModule proxy_connect_module modules/mod_proxy_connect.so
    LoadModule proxy_ftp_module modules/mod_proxy_ftp.so
    LoadModule proxy_http_module modules/mod_proxy_http.so
    <VirtualHost 10.0.2.15:443>
       SSLEnable
       SSLServerCert selfSigned
       ServerName portal.rural.gr
       ProxyRequests off
       ProxyPass /maximo https://maximo.rural.gr/maximo
       ProxyPassReverse /maximo https://maximo.rural.gr/maximo
    </VirtualHost>


Reconfigured db2 in maximo for 2GB because I'm running our of resources:
Ref: http://www-10.lotus.com/ldd/lfwiki.nsf/dx/Installing_And_Configuring_DB2_9.7_Server_And_The_Forms_Experience_Builder_Database
Inline image 1

Now magic: Nothing works, we get the page but all the html is reffering to the maximo.rural.gr instead of portal.rural.gr. We need to reverse proxy the html produced by maximo for the links as well.

Our IHS for 8.5.5 was doesn't support the mod_proxy_html module so I have to run another apache between the IHS of portal and the IHS of maximo. Let's configure this!

Download latest apache for windows 64 from: http://www.apachehaus.com/cgi-bin/download.plx?dli=TdFZWZ1QNFjT6J1KlRVUxAlVOpkVFVFdSBTVwk1d

Reverse proxy works but when you click a link it reverts to maximo.rural.gr even if we have a html/css/js rewriter enabled:

    LoadModule proxy_module modules/mod_proxy.so
    LoadModule proxy_connect_module modules/mod_proxy_connect.so
    LoadModule proxy_ftp_module modules/mod_proxy_ftp.so
    LoadModule proxy_html_module modules/mod_proxy_html.so
    LoadModule proxy_http_module modules/mod_proxy_http.so
    LoadModule rewrite_module modules/mod_rewrite.so
    LoadModule xml2enc_module modules/mod_xml2enc.so
    ServerName portal.rural.gr
    ProxyRequests Off
    ProxyHTMLEnable On
    SetOutputFilter  proxy-html
    ProxyHTMLExtended On
    ProxyHTMLURLMap http://maximo.rural.gr http://portal.rural.gr:12380
    ProxyPass /maximo http://maximo.rural.gr/maximo
    ProxyPassReverse /maximo http://maximo.rural.gr/maximo
    ProxyPass /webclient http://maximo.rural.gr/webclient
    ProxyPassReverse /webclient http://maximo.rural.gr/webclient
    RewriteEngine On
    RewriteRule ^/http://maximo.rural.gr/(.*) http://portal.rural.gr:12380/$1 [P,L]


Snooping the network is required now, maybe we need to rewrite protocol redirections of http as well, let's see where that gets us:

We run portal on windows so lookback interface snooping is not trivial like everything else with that OS, so:

    a. installed wireshark (loopback interface is not on windows by default)
    b. installed nmap-pcap to allow pcap local host interface to appear from: https://svn.nmap.org/nmap-exp/yang/NPcap-LWF/npcap-nmap-0.05.exe
    c. run cms to disable digital signing:
    bcdedit -set loadoptions DISABLE_INTEGRITY_CHECKS
    bcdedit -set TESTSIGNING ON
    d. reboot windows with f8 and boot with digital signing disabled


Now we can snoop without a problem our localhost apache that makes the reverse proxying to maximo without a problem.

And here comes our susspect:

    HTTP/1.1 200 OK
    Date: Mon, 12 Oct 2015 07:59:00 GMT
    Server: Apache/2.4.16 (Win64) OpenSSL/1.0.1p
    Cache-Control: no-store
    Expires: Thu, 01 Jan 1970 00:00:00 GMT
    Pragma: no-cache
    Content-Type: text/xml;charset=utf-8
    Content-Language: en-US
    Content-Length: 277
    Keep-Alive: timeout=5, max=100
    Connection: Keep-Alive
    <?xml version="1.0" encoding="UTF-8" ?>
    <server_response csrftoken="k4njtomrj3p1rc1j112b2phepk">
    <redirect><![CDATA[http://maximo.rural.gr/maximo/ui/?event=loadapp&value=startcntr&uniqueid=203&uisessionid=3&csrftoken=k4njtomrj3p1rc1j112b2phepk]]></redirect></server_response>

Maximo makes an ajax text/xml application redirect (not http 30x redirect). We need to figure a way pronto to mod proxy xml data as well on top of the others. 

While there is an experimental mod_proxy_xml from another party we will not use that. There is also a mod_sed module which is experimental as well but it doesn't do the trick for us. Finally I found mod_substitute module which is stable and with mod_filter module combination it can do what we want for our xml ajax requests:

Let's try that:

    LoadModule filter_module modules/mod_filter.so
    LoadModule substitute_module modules/mod_substitute.so
    AddOutputFilterByType SUBSTITUTE text/xml
    Substitute "s|maximo.rural.gr|portal.rural.gr|i"


Our final reverse proxy configuration at the end of httpd.conf of apache24:

    LoadModule proxy_module modules/mod_proxy.so
    LoadModule proxy_connect_module modules/mod_proxy_connect.so
    LoadModule proxy_ftp_module modules/mod_proxy_ftp.so
    LoadModule proxy_html_module modules/mod_proxy_html.so
    LoadModule proxy_http_module modules/mod_proxy_http.so
    LoadModule rewrite_module modules/mod_rewrite.so
    LoadModule xml2enc_module modules/mod_xml2enc.so
    LoadModule filter_module modules/mod_filter.so
    LoadModule substitute_module modules/mod_substitute.so
    ServerName portal.rural.gr
    ProxyRequests Off
    ProxyHTMLEnable On
    SetOutputFilter  proxy-html
    ProxyHTMLExtended On
    ProxyHTMLURLMap http://maximo.rural.gr http://portal.rural.gr:12380
    ProxyPass /maximo http://maximo.rural.gr/maximo
    ProxyPassReverse /maximo http://maximo.rural.gr/maximo
    ProxyPass /webclient http://maximo.rural.gr/webclient
    ProxyPassReverse /webclient http://maximo.rural.gr/webclient
    #RewriteEngine On
    #RewriteRule ^/http://maximo.rural.gr/(.*) http://portal.rural.gr:12380/$1 [P,L]
    AddOutputFilterByType SUBSTITUTE text/xml
    Substitute "s|http://maximo.rural.gr|http://portal.rural.gr:12380|n"

That works, which means that using an intermediary apache24 setup allows us to successfully reverse proxy the maximo product. We do that in order to trick javascript to think that we are on the same host port as portal. But portal runs on port 80 or 443 and not 12380. That's why we need to reverse proxy from 12380 to 443 in portal ihs server:

    <VirtualHost 10.0.2.15:443>
       SSLEnable
       SSLServerCert selfSigned
       ServerName portal.rural.gr
       ProxyRequests off
       ProxyPass /maximo http://portal.rural.gr:12380/maximo
       ProxyPassReverse /maximo http://portal.rural.gr:12380/maximo
       ProxyPass /webclient http://portal.rural.gr:12380/webclient
       ProxyPassReverse /webclient http://portal.rural.gr:12380/webclient
    </VirtualHost>


and because ihs doesn't support mod_proxy_html, mod_substitute, mod_filter we will also change the 12380 content substitutions to straight https://portal.rural.gr in order for this to make sense in our apache24 translator server:

    ServerName portal.rural.gr
    ProxyRequests Off
    ProxyHTMLEnable On
    SetOutputFilter  proxy-html
    ProxyHTMLExtended On
    #ProxyHTMLURLMap http://maximo.rural.gr http://portal.rural.gr:12380
    ProxyHTMLURLMap http://maximo.rural.gr https://portal.rural.gr
    ProxyPass /maximo http://maximo.rural.gr/maximo
    ProxyPassReverse /maximo http://maximo.rural.gr/maximo
    ProxyPass /webclient http://maximo.rural.gr/webclient
    ProxyPassReverse /webclient http://maximo.rural.gr/webclient
    #RewriteEngine On
    #RewriteRule ^/http://maximo.rural.gr/(.*) http://portal.rural.gr:12380/$1 [P,L]
    AddOutputFilterByType SUBSTITUTE text/xml
    #Substitute "s|http://maximo.rural.gr|http://portal.rural.gr:12380|n"
    Substitute "s|http://maximo.rural.gr|https://portal.rural.gr|n"

 
This way we keep ihs on the game and apache24 acts only as a reverse proxy only. In our case ihs also acts as an ssl terminator (if we wanted to do this).

Now https://portal.rural.gr/maximo works as a dream and we can also navigate without a problem. We need to check the apache24 for cpu usage doing all those subs for css/js/xml/html but this is not the time to do this.

Let's try to use a cookie to restore the last known navigation url in our iframe in order to avoid uisession leaks in maximo.

To do that in portal we used a simple content article and set the following source which consists of the javascript cookie navigation part and our iframe.

Here we will use docCookies stable cookie library with full unicode support to avoid any surprizes by handling maximo urls in cookies: https://developer.mozilla.org/en-US/docs/Web/API/document/cookie#A_little_framework.3A_a_complete_cookies_reader.2Fwriter_with_full_unicode_support

WebSphere Portal Content Article source code:

    <p dir="ltr">&nbsp;</p>
    <script type="text/javascript">
    var docCookies = {
      getItem: function (sKey) {
        if (!sKey) { return null; }
        return decodeURIComponent(document.cookie.replace(new RegExp("(?:(?:^|.*;)\\s*" + encodeURIComponent(sKey).replace(/[\-\.\+\*]/g, "\\$&") + "\\s*\\=\\s*([^;]*).*$)|^.*$"), "$1")) || null;
      },
      setItem: function (sKey, sValue, vEnd, sPath, sDomain, bSecure) {
        if (!sKey || /^(?:expires|max\-age|path|domain|secure)$/i.test(sKey)) { return false; }
        var sExpires = "";
        if (vEnd) {
          switch (vEnd.constructor) {
            case Number:
              sExpires = vEnd === Infinity ? "; expires=Fri, 31 Dec 9999 23:59:59 GMT" : "; max-age=" + vEnd;
              break;
            case String:
              sExpires = "; expires=" + vEnd;
              break;
            case Date:
              sExpires = "; expires=" + vEnd.toUTCString();
              break;
          }
        }
        document.cookie = encodeURIComponent(sKey) + "=" + encodeURIComponent(sValue) + sExpires + (sDomain ? "; domain=" + sDomain : "") + (sPath ? "; path=" + sPath : "") + (bSecure ? "; secure" : "");
        return true;
      },
      removeItem: function (sKey, sPath, sDomain) {
        if (!this.hasItem(sKey)) { return false; }
        document.cookie = encodeURIComponent(sKey) + "=; expires=Thu, 01 Jan 1970 00:00:00 GMT" + (sDomain ? "; domain=" + sDomain : "") + (sPath ? "; path=" + sPath : "");
        return true;
      },
      hasItem: function (sKey) {
        if (!sKey) { return false; }
        return (new RegExp("(?:^|;\\s*)" + encodeURIComponent(sKey).replace(/[\-\.\+\*]/g, "\\$&") + "\\s*\\=")).test(document.cookie);
      },
      keys: function () {
        var aKeys = document.cookie.replace(/((?:^|\s*;)[^\=]+)(?=;|$)|^\s*|\s*(?:\=[^;]*)?(?:\1|$)/g, "").split(/\s*(?:\=[^;]*)?;\s*/);
        for (var nLen = aKeys.length, nIdx = 0; nIdx < nLen; nIdx++) { aKeys[nIdx] = decodeURIComponent(aKeys[nIdx]); }
        return aKeys;
      }
    };
    function maximoLoaded() {
       var loc = document.getElementById("maxFrame").contentWindow.location.href;
       var url = loc.toString();
       // this will guarantee that event across logouts from sso the user returns to the point where he left, power...
       if (url=="" || url=="about:blank" || url.indexOf("login.jsp")!=-1 || url.indexOf("ibm_security_logout")) {
          //alert("url is blank or accessing login/logout pages, not setting cookie");
       } else { 

           // 3000 needs to be in sync with jsessionid timeout and uisession inactive timeout if we want to restore url to default splash page and don't want to have cross login navigation persistence

          docCookies.setItem("MAXNAVIURL",url,30000,"/","rural.gr");
          //alert("set cookie MAXNAVIURL=" + url);
       }
    }
    window.onload = function() {
    var url = docCookies.getItem("MAXNAVIURL");
       //alert("cookie read = " + url);
       if (url) {
          if (url=="about:blank" || url=="") {
            url="https://portal.rural.gr/maximo/ui/maximo.jsp?event=loadapp&value=sr&additionalevent=useqbe&portalmode=true";
          }
          //alert("url to restore is from cookie: " + url);
          document.getElementById('maxFrame').src = url;
       } else {
          var defaultUrl = "https://portal.rural.gr/maximo/ui/maximo.jsp?event=loadapp&value=sr&additionalevent=useqbe&portalmode=true";
          //alert("url to restore is default: " + defaultUrl );
          document.getElementById('maxFrame').src = defaultUrl;
       }
    }
    </script>
    <p dir="ltr"><iframe height="1000" id="maxFrame" name="ifr" onload="maximoLoaded()" width="1280"></iframe></p>


Using the above source for the article we can assure that iframe navigational state is sane and browsing across portal pages the user will return to the url that left before with the same uisessionid.

Let's restart maximo so as to login to it via portal run a few cycles and see if we get any problems with uisession leaks and logged-in users!

First /wps/portal screen, we have on purpose the aricle public to see that when we are logged out from portal we are also out from maximo:

Inline image 1

Login screen in portal:

Inline image 2

Welcome screen after loggin-in to portal:

Inline image 3

Bravely enough it brought us to the calendars were I left of last time! Great! Let's navigate to another portal page and come back to calendars:

Inline image 4

Back to maximo now:

Inline image 5

Great we are back in calendars, let's goto users and click manage sessions and see how many wpadmin users are logged in:

Inline image 6

That is a beauty we only have 1 user logged-in!!! No matter how many times I go back and forth to another portal page that doesn't general new ui sessions in maximo!

We still need to resolve 2 more things:

a. test maximo auto logout functionality redirect that doesn't though us out of portal:
Inline image 7
This is good because if we have synchronized sessionid timeouts when the user navigates to another portal page he will get this:
Inline image 8
b. test maximo protocol redirects when no other user sessions as allowed to login to maximo.

For search and replacing http redirects we can use mod_headers apache module having in mind this:

http://httpd.apache.org/docs/current/mod/mod_headers.html#header
http://stackoverflow.com/questions/16297233/how-to-rewrite-location-response-header-in-a-proxy-setup-with-apache

like that:

Header edit Location ^http://maximo.rural.gr/(.*)$ https://portal.rural.gr/$1

Header edit Location ^https://maximo.rural.gr/(.*)$ https://portal.rural.gr/$1

Now we need to put this to our apache24 translator server and cause an http redirect.

To cause a 30x redirect we need to limit maximo ui sessions up to 2 and login twice externally from portal and then try to access portal login process.

Updated maximo system props:

    mxe.webclient.maxuisessions=2 (defaults to 0)
    mxe.webclient.maxuisessionsend503=0 (defaults to 1)


Here we try to avoid in reality 2 kind of redirects, the 503 (which I have disabled) and the 30x which you cannot.

Let's see the results going straight to maximo when the uisessions are full and the server cannot hold any more:

Inline image 9

As we see we get a 302 redirect to a page saying that we have too many users in maximo. We need to run this via portal and examine its behaviour:

Inline image 10

This now works like a charm. We are covered, http redirections are translated perfectly and we always stay within portal.

##CONCLUSION

Architecturally, having an apache latest running next to IHS will provide all the translation and reverse proxy required in order to run maximo next to portal under the same domain and avoid security browser issues. That allows as to use javascript and a cookie to store the last known maximo location that contains the uisessionid and not leak users in maximo leading to resource exaustion or DOS. We need to keep apache next to IHS to allow IHS to run the was plugin.

Attention needs to be given on the inactivity timers (or session timeouts), it seems that when we logout from portal the user is not getting logged out from maximo successfully. He will get out eventually but after the inactivity of ui session kicks in. We can resolve this using a portal logout hook module which might gives us access to the http session and cookies and run an artificial logout when portal logout is in progress. 

Further things to be done to consider this bullet proof:

1. Install CTC in portal and run layout tests.
2. Run maximo coverage tests in order to CRUD in all major apps of interest like WO, SR, LOCATIONS, ASSETS, CLASSIFICATIONS etc. 

That's it for now!
