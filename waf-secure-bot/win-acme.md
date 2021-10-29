````cmd
A simple Windows ACMEv2 client (WACS)
Software version 2.1.19.1142 (release, pluggable, standalone, 64-bit)
Connecting to https://acme-v02.api.letsencrypt.org/...
Scheduled task not configured yet
Please report issues at https://github.com/win-acme/win-acme

N: Create certificate (default settings)
M: Create certificate (full options)
R: Run renewals (0 currently due)
A: Manage renewals (0 total)
O: More options...
Q: Quit

Please choose from the menu: M

Running in mode: Interactive, Advanced

Please specify how the list of domain names that will be included in the
certificate should be determined. If you choose for one of the "all bindings"
options, the list will automatically be updated for future renewals to
reflect the bindings at that time.

1: Read site bindings from IIS
2: Manual input
3: CSR created by another program
C: Abort

How shall we determine the domain(s) to include in the certificate?: 2

Description:        A host name to get a certificate for. This may be a
                    comma-separated list.

Host: echo-bot.techaholics.gr

Source generated using plugin Manual: echo-bot.techaholics.gr

Suggested friendly name '[Manual] echo-bot.techaholics.gr', press <Enter> to accept or type an alternative: <Enter>

The ACME server will need to verify that you are the owner of the domain
names that you are requesting the certificate for. This happens both during
initial setup *and* for every future renewal. There are two main methods of
doing so: answering specific http requests (http-01) or create specific dns
records (dns-01). For wildcard domains the latter is the only option. Various
additional plugins are available from https://github.com/win-acme/win-acme/.

1: [http-01] Save verification files on (network) path
2: [http-01] Serve verification files from memory
3: [http-01] Upload verification files via FTP(S)
4: [http-01] Upload verification files via SSH-FTP
5: [http-01] Upload verification files via WebDav
6: [dns-01] Create verification records manually (auto-renew not possible)
7: [dns-01] Create verification records with acme-dns (https://github.com/joohoi/acme-dns)
8: [dns-01] Create verification records with your own script
9: [tls-alpn-01] Answer TLS verification request from win-acme
C: Abort

How would you like prove ownership for the domain(s)?: 6

After ownership of the domain(s) has been proven, we will create a
Certificate Signing Request (CSR) to obtain the actual certificate. The CSR
determines properties of the certificate like which (type of) key to use. If
you are not sure what to pick here, RSA is the safe default.

1: Elliptic Curve key
2: RSA key
C: Abort

What kind of private key should be used for the certificate?: 2

When we have the certificate, you can store in one or more ways to make it
accessible to your applications. The Windows Certificate Store is the default
location for IIS (unless you are managing a cluster of them).

1: IIS Central Certificate Store (.pfx per host)
2: PEM encoded files (Apache, nginx, etc.)
3: PFX archive
4: Windows Certificate Store
5: No (additional) store steps

How would you like to store the certificate?: 3

Description:        Path to write the .pfx file to.

File path: c:\temp

Description:        Password to set for .pfx files exported to the folder.

1: None
2: Type/paste in console
3: Search in vault

Choose from the menu: 2

PfxPassword: ****

Save to vault for future reuse? (y/n*) - no

1: IIS Central Certificate Store (.pfx per host)
2: PEM encoded files (Apache, nginx, etc.)
3: PFX archive
4: Windows Certificate Store
5: No (additional) store steps

Would you like to store it in another way too?: 5

Installation plugin IIS not available: This step cannot be used in combination with the specified store(s)

With the certificate saved to the store(s) of your choice, you may choose one
or more steps to update your applications, e.g. to configure the new
thumbprint, or to update bindings.

1: Create or update https bindings in IIS
2: Create or update ftps bindings in IIS
3: Start external script or program
4: No (additional) installation steps

Which installation step should run first?: 4

Terms of service:   C:\ProgramData\win-acme\acme-v02.api.letsencrypt.org\LE-SA-v1.2-November-15-2017.pdf

Open in default application? (y/n*) - no

Do you agree with the terms? (y*/n) - yes

Enter email(s) for notifications about problems and abuse (comma-separated): xxx@xxx.xxx

[echo-bot.techaholics.gr] Authorizing...
[echo-bot.techaholics.gr] Authorizing using dns-01 validation (Manual)

Domain:             echo-bot.techaholics.gr
Record:             _acme-challenge.echo-bot.techaholics.gr
Type:               TXT
Content:            "nPRUg-SoWCrZx_dKJ86Aoy0BDOclNg5wgydNyZlQuXo"
Note:               Some DNS managers add quotes automatically. A single set
                    is needed.

Please press <Enter> after you've created and verified the record
````