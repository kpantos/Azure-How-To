# Web Farm SPN Configuration

By default, a web server running Internet Information Server (IIS) versions 7, 7.5, and 8 uses the kernel-mode feature for Windows authentication, which is enabled by default. This authentication does not depend on the application pool identity to decrypt incoming requests, instead using the machine account (Local system) of the IIS server. This means that, for IIS 7, 7.5, and 8 using default settings, an HTTP SPN registration for the application pool identity is not required.

If you are implementing a web farm, you need greater control over identity. So, you will need to override this default behavior to make sure that the web farm identity (which relates to the web farm's SPN) is the same for all nodes. This can be achieved by using the useAppPoolCredentials="true" setting for Windows authentication (see https://technet.microsoft.com/en-au/library/dd573004%28v=office.13%29.aspx) to revert to the IIS 6 behavior of requiring an application pool identity. In this case, you will require an SPN for the web farm alias with the application pool identity.

## Using setspn.exe to Create and Manage SPNs
```setspn.exe``` must be executed from the command prompt.

### Examples
The following table includes examples for WebServer01 and WebServiceUser01 in the Contoso domain.

<table>
    <thead>
        <tr>
            <th width="40%">Example</th>
            <th>Command</th>
        </tr>
    </thead>
    <tr>
        <td>List the SPN registrations for a computer account.</td>
        <td><code>setspn -L Contoso\WebServer01</code></td>
    </tr>
    <tr>
        <td>List the SPN registrations for a service user account.</td>
        <td><code>setspn –L Contoso\WebServiceUser01</code></td>
    </tr>
    <tr>
        <td rowspan="4">Add HTTP and HTTPS registrations to WebServiceUser01 because the Application Pool is running as this domain user.</td>
        <td><code>setspn -A HTTP/WebServer01 Contoso\WebServiceUser01</code></td>
    </tr>      
    <tr>
        <td><code>setspn –A HTTP/WebServer01.Contoso.com Contoso\WebServiceUser01</code></td>
    </tr>
    <tr>
        <td><code>setspn -A HTTPS/WebServer01 Contoso\WebServiceUser01</code></td>
    </tr>
    <tr>
        <td><code>setspn –A HTTPS/WebServer01.Contoso.com Contoso\WebServiceUser01</code></td>
    </tr>
    <tr>
        <td rowspan="4">Add HTTP and HTTPS registrations for any DNS aliases to WebServer01 if it is running as NETWORK SERVICE.</td>
        <td><code>setspn –A HTTP/zapbiDnsAlias Contoso\WebServer01</code></td>
    </tr>
        <tr>
        <td><code>setspn –A HTTP/zapbiDnsAlias.Contoso.com Contoso\WebServer01</code></td>
    </tr>
        <tr>
        <td><code>setspn –A HTTPS/zapbiDnsAlias Contoso\WebServer01</code></td>
    </tr>
        <tr>
        <td><code>setspn –A HTTPS/zapbiDnsAlias.Contoso.com Contoso\WebServer01</code></td>
    </tr>
    <tr>
        <td rowspan="4">Add HTTP and HTTPS registrations for any DNS alias to WebServiceUser01 if it is running as the WebServiceUser01 user account.</td>
        <td><code>setspn –A HTTP/zapbiDnsAlias Contoso\WebServiceUser01</code></td>
    </tr>
        <tr>
        <td><code>setspn –A HTTP/zapbiDnsAlias.Contoso.com Contoso\WebServiceUser01</code></td>
    </tr>
        <tr>
        <td><code>setspn –A HTTPS/zapbiDnsAlias Contoso\WebServiceUser01</code></td>
    </tr>
        <tr>
        <td><code>setspn –A HTTPS/zapbiDnsAlias.Contoso.com ZAP\WebServiceUser01</code></td>
    </tr>
</table>
