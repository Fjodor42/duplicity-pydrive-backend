# duplicity-pydrive-backend

Pydrive backend is merged now in the development version of duplicity, so better use it from there.

Currently duplicity uses gdocs backend for Google Drive backups. gdocs uses deprecated API and don't allow backups for managed Google accounts. PyDrive backend solves both of those problems.

## Install dependecies
```
apt-get build-dep duplicity
apt-get install python-crypto python-setuptools checkinstall python-lockfile
```
## Generate gpg keys
```
gpg --gen-key
```
## Create a Google project to to act on your behalf

As stated above, PyDrive enables the use of managed Google accounts through a service account, and the instructions below will explain the use of such ones.

In the case of a single Google account with a large Drive, though, you might not want or be able to use that approach, and should use the following procedure instead

### Create Google project to act on your behalf
- Go to [Google developers console](https://console.developers.google.com) and create a project
- Under APIs, find the "Drive API" and enable it
- Under Credentials, click "Create new Client ID"
- For "Application type", select "Installed application"
- For "Installed application type", select "Other" and click "Create Client ID"
- Click "Download JSON", and save the file under the name "client_secrets.json" in an appropriate location
- Make sure to do the following in the script you run duplicity from:
  -
```
export GOOGLE_AUTH_MODE=private
```
  -
```
export GOOGLE_SECRETS_FILE=/some/path/to/client_secrets.json
```
  -
```
export GOOGLE_CREDENTIALS_FILE=/some/path/to/client_credentials.json
```
  -
```
duplicity <command> <args> pydrive://user@gmail.com/relevant_drive_folder
```
  -
```
unset GOOGLE_AUTH_MODE
```
  -
```
unset GOOGLE_SECRETS_FILE
```
  -
```
unset GOOGLE_CREDENTIALS_FILE
```

### Create service account in your Google account
- Go to [Google developers console](https://console.developers.google.com) and create a service account
- Write down the email address of the account, XXX@developer.gserviceaccount.com 
- Download a .p12 key file, then convert it to the .pem:
```
openssl pkcs12 -in XXX.p12  -nodes -nocerts > pydriveprivatekey.pem
```
(you'll need a password you got when created the account)
- 
- Make sure to do the following in the script you run duplicity from:
  - ```
export GOOGLE_AUTH_MODE=managed
```
  -
```
export GOOGLE_DRIVE_ACCOUNT_KEY="<contents of pydriveprivatekey.pem
including
line
breaks
and
a
final
quotation
mark"
```
--
```
duplicity <command> <args> pydrive://XXX@developer.gserviceaccount.com/relevant_drive_folder
```
--
```
unset GOOGLE_AUTH_MODE
```
--
```
unset GOOGLE_DRIVE_ACCOUNT_KEY
```
--```
unset GOOGLE_CREDENTIALS_FILE
```

## Download, patch and install duplicity
### Download duplicity and signature file
```
wget https://launchpad.net/duplicity/0.7-series/0.7.00/+download/duplicity-0.7.0.tar.gz
wget https://launchpad.net/duplicity/0.7-series/0.7.00/+download/duplicity-0.7.0.tar.gz.sig 
```
### Verify downloaded file
```
gpg --verify duplicity-0.7.0.tar.gz.sig 
```
If you don't have duplicity developers' key, i.e. getting this message:
```
gpg: Signature made Thu 23 Oct 2014 04:19:49 PM IDT using DSA key ID E654E600
gpg: Can't check signature: public key not found
```
Fetch the key:
```
gpg --recv-keys E654E600
gpg: requesting key E654E600 from hkp server keys.gnupg.net
gpg: key E654E600: public key "Duplicity Test (no password) <duplicity@loafman.com>" imported
gpg: Total number processed: 1
gpg:               imported: 1
```
Then try to verify again:
```
gpg --verify duplicity-0.7.0.tar.gz.sig 
gpg: Signature made Thu 23 Oct 2014 04:19:49 PM IDT using DSA key ID E654E600
gpg: Good signature from "Duplicity Test (no password) <duplicity@loafman.com>"
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 9D95 920C ED4A 8D5F 8B08  6A9F 8B6F 8FF4 E654 E600
```
### Patch
```
tar -xzvf duplicity-0.7.0.tar.gz
cd /duplicity-0.7.0
```
#### Edit duplicity/backend.py file:
1 Find this:
```
uses_netloc = ['ftp',
               'ftps',
               'hsi',
               's3',
               'scp', 'ssh', 'sftp',
               'webdav', 'webdavs',
               'gdocs',
               'http', 'https',
               'imap', 'imaps',
               'mega',
               'copy']
```
And change to this (add "pydrive" to the end of the list):
```
uses_netloc = ['ftp',
               'ftps',
               'hsi',
               's3',
               'scp', 'ssh', 'sftp',
               'webdav', 'webdavs',
               'gdocs',
               'http', 'https',
               'imap', 'imaps',
               'mega',
               'copy',
               'pydrive']
```
2 Find this:
```
try:
    pu = urlparse.urlparse(url_string)
except Exception:
    raise InvalidBackendURL("Syntax error in: %s" % url_string)
```
And right after that add:
```
if pu.query:     
    try:
        self.keyfile = urlparse.parse_qs(pu.query)['keyfile']
    except Exception:
        raise InvalidBackendURL("Syntax error (keyfile) in: %s" % url_string)
```
On the same indent level
#### Fetch pydrivebackend.py from here and add it to the backends folder:
```
wget https://github.com/westerngateguard/duplicity-pydrive-backend/blob/master/pydrivebackend.py
mv pydrivebackend.py duplicity-0.7.0/duplicity/backends/
```
#### Finally, build (as root):
```
checkinstall python setup.py install
```
## Install PyDrive (as root)
```
apt-get install python-yaml python-simplejson python-pip
checkinstall pip install PyDrive
```
## Use the backend by this remote URL:
```
duplicity full --encrypt-key *YOUR_gpg_KEY* --sign-key *YOUR_gpg_KEY* /local/folder/path pydrive://XXX@developer.gserviceaccount.com/remote/path/on/google/drive?keyfile=/path/to/your/pydriveprivatekey.pem
```
