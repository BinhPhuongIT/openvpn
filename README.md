# Open VPN with Google Authen

docker pull binhphuong/openvpn:1.0.0
cd ..
mkdir vpn-data && touch vpn-data/vars

# Generating OpenVPN config file
# Choose a more secure cipher to use because since OpenVPN 2.3.13 the default openvpn cipher BF-CBC will cause a renegotiated connection every 64 MB of data
# Generate server configuration with -2 and -C $CIPHER options
# https://community.openvpn.net/openvpn/wiki/SWEET32
docker run -v $PWD/vpn-data:/etc/openvpn --rm binhphuong/openvpn:1.0.0 ovpn_genconfig -u udp://vpn.example.com:3000 -2 -C 'AES-256-CBC' -a 'SHA512'

# Generating our CA certificate and we will have a private key belong to the PKI
docker run -e EASYRSA_KEY_SIZE=4096 -v $PWD/vpn-data:/etc/openvpn --rm -it binhphuong/openvpn:1.0.0 ovpn_initpki

# Run the VPN server based on that config
# User command
docker run -v $PWD/vpn-data:/etc/openvpn -d --name openvpn -p 3000:1194/udp --cap-add=NET_ADMIN binhphuong/openvpn:1.0.0
# User Compose
docker-compose up -d openvpn

# Create user
docker run -e EASYRSA_KEY_SIZE=4096 -v $PWD/vpn-data:/etc/openvpn --rm -it binhphuong/openvpn:1.0.0 easyrsa build-client-full user1 nopass

# Generate authentication configuration for your client. -t is needed to show QR code, -i is optional for interactive usage
docker run -v $PWD/vpn-data:/etc/openvpn --rm -it binhphuong/openvpn:1.0.0 ovpn_otp_user user1

# Get client config
docker run -v $PWD/vpn-data:/etc/openvpn --rm binhphuong/openvpn:1.0.0 ovpn_getclient user1 > user1.ovpn

# Get client QR code
docker run -v $PWD/vpn-data:/etc/openvpn --rm binhphuong/openvpn:1.0.0 google-authenticator --time-based --disallow-reuse --force --rate-limit=3 --rate-time=30 --window-size=3 \
-l "${1}@${OVPN_CN}" -s /etc/openvpn/otp/${1}.google_authenticator


##############################
# remove client
docker run -it -v $PWD/vpn-data:/etc/openvpn --rm binhphuong/openvpn:1.0.0 ovpn_revokeclient client1
