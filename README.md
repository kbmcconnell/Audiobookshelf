# Audiobookshelf on Raspberry Pi 5
Below are the resources and steps I followed to set up an Audiobookshelf library on a Raspberry Pi 5 using Cloudflared and Docker. This is not a troubleshooting guide and the configuration might not suit your needs. To make my library available over the web, I opted to use a Cloudflared tunnel with a custom domain in lieu of a reverse proxy.

## Technology & Services used
- [Audiobookshelf](https://www.audiobookshelf.org/)
- Raspberry Pi 5
  - running the latest version of Bookworm OS
- [Cloudfared tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) with custom domain
- [Docker](https://www.docker.com/) containers

## 1. Update your Raspberry Pi
First make sure existing packages on your Raspberry Pi are up to date.<br>
```
sudo apt update
sudo apt upgrade -y
```

## 2. Install & configure Docker
### Installation
Details about the script can be found [here](https://get.docker.com/).

Install Docker:
```
curl -sSL https://get.docker.com | sh
```
If you don't have `curl` installed, install it:
```
sudo apt install curl
```

To interact with Docker you need to add your current user to the docker group, otherwise you can only interact with Docker when you're running as the root user. You can learn more about Linux permissions [here](https://medium.com/codex/users-groups-and-permissions-in-linux-93895ae57d93).<br><br>
Add current user to docker group:
```
sudo usermod -aG docker $USER
```

Log out and back in to apply changes:
```
logout
su <username>
```

After logging back in, verify the docker group has been added to your user. Type `groups` in the terminal - this will print out all of the groups your current user is a part of. Look for "docker" in the list.

### Optional: Test Docker installation
Test that you successfully installed Docker by running the following command:
```
docker run hello-world
```
You will see text output from Docker which confirms that it's set up and running successfully.

### Configuration
Next, create a directory where your Audiobookshelf metadata will live:
```
sudo mkdir -p /opt/stacks/audiobookshelf
```

Navigate to the directory you just created for the remaining steps:
```
cd /opt/stacks/audiobookshelf
```

## 3. Create Docker Compose file
To see an example of a completed Docker Compose file, head here. See [Audiobookshelf's Docker Compose](https://www.audiobookshelf.org/docs#docker-compose-install) instructions for additional details:
```
sudo nano compose.yaml
```
Example:
```
services:
  audiobookshelf:
    image: ghcr.io/advplyr/audiobookshelf:latest
    ports:
      - 13378:80
    volumes:
      - /home/kbmcconnell/audiobooks:/audiobooks
      - /home/kbmcconnell/podcasts:/podcasts
      - .config:/config
      - .metadata:/metadata
    environment:
      - TZ=America/Los_Angeles
```

Once you've filled out your compose file, save and quit.

## 4. Set up Audiobookshelf
Let's get Audiobookshelf set up before moving to the next steps. Use "-d" in your command so Docker detaches from the terminal once it starts:
```
docker compose up -d
```
Get your Raspberry Pi's IP address by running the command `hostname -i` in the terminal. Go to your Pi's web browser and enter the following address:
```
http://<IP.ADDRESS>:13378
```
You should be greeted by the Audiobookshelf user creation page for the root user. Create the root user for your Audiobookshelf account. You will be prompted to log in once you've created your account. Refer to Audiobookshelf's documentation for additional configuration and directory structure.

## 5. Set up Cloudflared tunnel
If you wish to have access to your Audiobookshelf library from anywhere, or to share access with friends and family, you will need to decide how to set it up. I use Cloudflared so below are instructions for that. If you choose to use a reverse proxy instead, there are a lot of examples available online to guide you.

### Installation
Update your packages if you're doing this some time after your initial Audiobookshelf setup:
```
sudo apt update
sudo apt upgrade
```
Install `curl` and `lsb-release` if they aren't already installed:
```
sudo apt install curl lsb-release
```
Run the following command to grab the GPG key for the Cloudflared repo and store it on the Raspberry Pi. You must do this to ensure the packages you're installing belong to the repo:
```
curl -L https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-archive-keyring.gpg >/dev/null
```
Next, add the Cloudflared repo to your Pi:
```
echo "deb [signed-by=/usr/share/keyrings/cloudflare-archive-keyring.gpg] https://pkg.cloudflare.com/cloudflared $(lsb_release -cs) main" | sudo tee  /etc/apt/sources.list.d/cloudflared.list
```
Update your packages again:
```
sudo apt update
```
Install Cloudflared:
```
sudo apt install cloudflared
```

### Connect to Cloudflare service
Authenticate with Cloudflare to connect to your Raspberry Pi:
```
cloudflared tunnel login
```
After running this command, you should see a url in your terminal that you need to visit to login with your Cloudflare account (or create an account). When you've successfully authenticated with your Raspberry Pi, you should see a message like this:
```
You have successfully logged in.
If you wish to copy your credentials to a server, they have been saved to:
/home/kbmcconnell/.cloudflared/cert.pem
```
### Create Cloudflared tunnel
Once you've authenticated, create your tunnel:
```
cloudflared tunnel create TUNNELNAME
```
You should see a confirmation that your tunnel was successfully created:
```
Tunnel credentials written to /home/kmbmcconnell/.cloudflared/XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX.json. cloudflared chose this file based on where your origin certificate was found. Keep this file secret. To revoke these credentials, delete the tunnel.

Created tunnel TUNNELNAME with id XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
```
In the Cloudflare Zero Trust [dashboard](https://one.dash.cloudflare.com/), under Network > Tunnels, you should now see the tunnel you created.

### Set up custom domain
To purchase and/or register a new domain, go to your Cloudflare [dashboard](https://dash.cloudflare.com/) then Domain Registration > Register Domains. For convenience, you can purchase a Universal SSL certificate through Cloudflare.

Next, create a DNS record for your tunnel. More detailed information can be found [here](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/routing-to-tunnel/dns/):
```
cloudflared tunnel route dns TUNNELNAME yourdomain.com
```
If this was successful, you will see a confirmation similar to this:
```
INF Added CNAME yourdomain.com which will route to this tunnel tunnelID=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
```
Huzzah! Your tunnel should now be linked to your domain.

### Set up config file
Set up a config file for your tunnel.
```
sudo nano ~/.cloudflared/config.yml
```
Here's an example configuration:
```
tunnel: TUNNELNAME
credentials-file: /home/kbmcconnell/.cloudflared/tunnel-id.json
ingress:
  - hostname: yourdomain.com
    service: http://localhost:13378
  - service: http_status:404
```
**Note:** you may need to install cloudflared as a systemctl service with this configuration. You do not need to do this if you're not experiencing issues accessing your domain. If you're running into problems, try the following:
```
sudo cloudflared --config ~/.cloudflared/config.yml service install
sudo systemctl enable cloudflared
```
Start the tunnel:
```
sudo systemctl start cloudflared
```
Try `docker compose up -d` again after setting up the service if you've completed all of the other steps.

## Put it all together
In order for your Audiobookshelf and Cloudflared containers to talk to each other, you need to create a network. Documentation can be found [here](https://docs.docker.com/reference/cli/docker/network/create/):
```
docker network create -d bridge my-network
```
You must add this information to your Docker compose file which will look something like this:
```
networks:
  my-network:
    driver: bridge
```
Now update your Docker compose file to include the cloudflared service and network you've created:
```
services:
  audiobookshelf:
    image: ghcr.io/advplyr/audiobookshelf:latest
    ports:
      - 13378:80
    volumes:
      - /home/kbmcconnell/audiobooks>:/audiobooks
      - /home/kbmcconnell/podcasts:/podcasts
      - ./config:/config
      - ./metadata:/metadata
    environment:
      - TZ=America/Los_Angeles
    restart: unless-stopped
    networks:
      - my-network
  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    command: tunnel run --url localhost:13378 --token TUNNELNAME
    volumes:
      - /cloudflared:/home/kbmcconnell/.cloudflared
    networks:
      - my-network
    restart: unless-stopped

networks:
  my-network:
    driver: bridge
```
Since you made changes to the Docker file you need to pull the changes:
```
docker compose pull
```
The last thing left to do is to run your docker compose command to spin up your containers:
```
docker compose up -d
```
You should see a confirmation that both of your containers have been created. Test the setup was successful by visiting your domain. You should be greeted with the Audiobookshelf login page.

## Cloudflare settings recommendations
Here are some basic recs to keep your Audiobookshelf secure. Feel free to do way more than this. These are all available on the free plan.
**SSL/TLS settings**
- Universal SSL Edge certificate
- SSL/TLS encryption: full (strict)
- Always Use HTTPS: toggle on
- Opportunistic Encryption: toggle on
- TLS 1.3: toggle on
- Automatic HTTPS Rewrites: toggle on

## Resources
- [Audiobookshelf](https://www.audiobookshelf.org/
- [Docker CLI documentation](https://docs.docker.com/reference/)
- [Cloudflare tunnel documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
