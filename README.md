<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <system.webServer>
    <modules>
      <remove name="WebDAVModule" />
    </modules>
    <handlers>
      <add name="Python FastCGI" path="*" verb="*" modules="FastCgiModule" scriptProcessor="E:\apps\SADRD\vSadrdEnv\python.exe|E:\apps\SADRD\vSadrdEnv\Lib\site-packages\wfastcgi.py" resourceType="Unspecified" requireAccess="Script" />
    </handlers>
    <directoryBrowse enabled="true" />
    <rewrite>
      <outboundRules rewriteBeforeCache="false">
        <rule name="Remove Server header">
          <match serverVariable="RESPONSE_Server" pattern=".+" />
          <action type="Rewrite" value="" />
        </rule>
      </outboundRules>
    </rewrite>
    <httpProtocol>
      <customHeaders>
        <remove name="X-Powered-By" />
        <remove name="X-ASPNet-Version" />
      </customHeaders>
    </httpProtocol>
  </system.webServer>
  <appSettings>
    <!-- Required settings -->
    <add key="WSGI_HANDLER" value="APIHome.mainapp" />
    <add key="PYTHONPATH" value="E:\apps\SADRD\vSadrdEnv" />
    <add key="FLASK_ENV" value="production" />
    <add key="WSGI_LOG" value="E:\apps\SADRD\Logs\wfastcgi.log" />
  </appSettings>
</configuration>
