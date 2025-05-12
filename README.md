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
Details about the script can be found [here](https://get.docker.com/).<br><br>
Install Docker:<br>
```
curl -sSL https://get.docker.com | sh
```
If you don't have `curl` installed, install it:
```
sudo apt install curl
```

To interact with Docker you need to add your current user to the docker group, otherwise you can only interact with Docker when you're running as the root user. You can learn more about Linux permissions [here](https://medium.com/codex/users-groups-and-permissions-in-linux-93895ae57d93).<br><br>
Add current user to docker group:<br>
```
sudo usermod -aG docker $USER
```

Log out and back in to apply changes:<br>
```
logout
su <username>
```

After logging back in, verify the docker group has been added to your user. This will print out all of the groups your current user is a part of - look for "docker" in the list.<br>
```
groups
```

### Optional: Test Docker installation
Test that you successfully installed Docker by running the following command:<br>
```
docker run hello-world
```
You will see text output from Docker which confirms that it's set up and running successfully.

### Configuration
Next, create a directory where your Audiobookshelf metadata will live.<br>
```
sudo mkdir -p /opt/stacks/audiobookshelf
```

Navigate to the directory you just created.<br>
```
cd /opt/stacks/audiobookshelf
```

## 3. Create Docker Compose file
To see an example of a completed Docker Compose file, head here. See [Audioshelf's Docker Compose](https://www.audiobookshelf.org/docs#docker-compose-install) instructions for additional details.<br>
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

Once you've filled out your Compose file, save and quit.

## 4. Set up Audiobookshelf
Let's get Audiobookshelf set up before moving to the next steps. Use "-d" in your command so Docker detach from the terminal once it starts.
```
docker compose up -d
```
Get your Raspberry Pi's IP address by running the command `hostname -i` in the terminal. Go to your Pi's web browser and enter the following address:
```
http://<IP.ADDRESS>:13378
```
Create a root user for your Audiobookshelf account. You will be prompted to log in once you've created your account. Refer to Audiobookshelf's documentation for additional configuration and directory structure.

## 5. Cloudflared tunnel
If you wish to have access to your Audiobookshelf library from anywhere, or to share access with friends and family, you will need to decide how to set it up. I use Cloudflared so below are instructions for that. If you choose to use a reverse proxy instead, there are a lot of examples available online to guide you.

Update your packages if you're doing this some time after your initial Audiobookshelf setup:
```
sudo apt update
sudo apt upgrade
```
