# Setting up synapse matrix server on a VPS

## Using podman (docker) image. HTTPS (self signed cert)
**IMPORTANT NOTE : Values into [Brackets] must be modified with your values without the [ ] characters.**

### Prerequisites
0. Install podman, activate podman service as user using `systemctl start --user podman` (if not already started).
0. Set up your server to be reachable via both IPV4/IPV6 with a domain name (OVH VPS dont listen to IPV6 by default for instance, so check it)

### setup
1. get synapse docker image using `podman pull docker.io/matrixdotorg/synapse:latest`
2. run `podman image list` and note the **IMAGE ID** of the pulled image
3. run image for initial config file generation using `podman run -it -p --rm --mount type=volume,src=synapse-data,dst=/data -e SYNAPSE_SERVER_NAME=[SERVER_NAME] -e SYNAPSE_REPORT_STATS=no [IMAGE_ID] generate` :
 - **[SERVER_NAME]** being the domain name of your server ( ex: youteub.in )
 - **[IMAGE_ID]** being the IMAGE ID you grabbed before
   This will create the volume **synapse-data** in your podman environment
4. verify the volume has been created with `podman volume ls` **synapse-data** should be in the list.
5. First use `podman unshare` to log as root and unlock volume edit.
6. Then type  `podman volume mount synapse-data`. The response should give the path to the volume, ex : `~/.local/share/containers/storage/volumes/synapse-data/_data`
7. `cd ~/.local/share/containers/storage/volumes/synapse-data/_data && nano homserver.yaml`
8. Put as follow:
```
server_name: "[SERVER_NAME]"
pid_file: /data/homeserver.pid
listeners:
  - port: 8448
    type: http
    tls: true
    resources:
      - names: [client, federation]
        compress: false
tls_certificate_path: "/data/cert.pem"
tls_private_key_path: "/data/key.pem"
database:
  name: sqlite3
  args:
    database: /data/homeserver.db
log_config: "/data/[SERVER_NAME].log.config"
media_store_path: /data/media_store
registration_shared_secret: "dzerezrezfzefzefezfezf"
enable_registration: true
enable_registration_without_verification: true
report_stats: false
macaroon_secret_key: "zefezfzefsdfdfezfezfezfze"
form_secret: "fezfezfezfezfezfzefezfzfzef"
signing_key_path: "/data/[SERVER_NAME].signing.key"
trusted_key_servers:
  - server_name: "[SERVER_NAME]"
```
 - **[SERVER_NAME]** being the domain name of your server (some of the fields will be automatically generated, so only change those indicated)
 - **Note:** For troubleshooting you can set `tls: false` and omit `tls_certificate_path` & `tls_private_key_path` entries. Port can also be set to 8008 with `port: 8008`
9. Generate SSL keys using `openssl req -newkey rsa:2048 -nodes -keyout key.pem -x509 -days 365 -out cert.pem`. Dont forget to put the [SERVER_NAME] when the cert creation will ask for FQDN.Rest can be bullshit.
10. Change ownership of files using `chown 991 key.pem && chown 991 cert.pem` . Used 991 as it was owner of other files in directory.
11. exit root session using `exit`

### Running server
1. Run image using `podman run -it -p 8448:8448 --rm --mount type=volume,src=synapse-data,dst=/data [IMAGE_ID]`
 - **[IMAGE_ID]** being the IMAGE ID you grabbed before
 - If using troubleshooting settings, use `-p 8008:8008` instead of `-p 8448:8448`
2. check with your internet browser `https://[SERVER_NAME]:8448` ( or `http://[SERVER_NAME]:8008` if using troubleshooting settings) . Normally a page shoud display indicating the server is up and running.
3. Go to `https://app.element.io/#/login`.
4. Change the homeserver from `matrix.org` to `[SERVER_NAME]:8448`. Click continue.
At this point the HTML client will make requests to your server so check for errors if the webapp doesn't accept it.
5. You should see the homeserver updated to `https://[SERVER_NAME]:8448`
6. Create user/password. Be careful that no limit is yet in place for user to create accounts.Consider editing `enable_registration` and `enable_registration_without_verification` config file fields further.
7. You should now be connected the server!! :)

## Sources
- https://github.com/element-hq/synapse
- https://www.youtube.com/watch?v=aeps4cicDoI
