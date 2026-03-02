You can safely disable unused PHP-FPM versions in CloudPanel. Based on my testing, here's the safest approach:

1. Stop and disable the PHP-FPM services, but do not delete or rename the files:

```bash
# For versions you're not using (e.g., PHP 7.1, 7.2, 7.3, 7.4):
systemctl stop php7.1-fpm
systemctl disable php7.1-fpm
# Repeat for other unused versions



--

You will need to take care of NAT HAIRPIN on HIGH END SERVERS

```bash
echo "127.0.0.1 windshear.fr www.windshear.fr radar.windshear.fr www.radar.windshear.fr" | sudo tee -a /etc/hosts
echo "127.0.0.1 windshear.fr www.windshear.fr radar.windshear.fr www.radar.windshear.fr" | sudo tee -a /etc/cloud/templates/hosts.debian.tmpl
```