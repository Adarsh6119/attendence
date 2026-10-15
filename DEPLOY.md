# Existing Hostinger VPS par Time Tracker add karna

Yeh guide us VPS ke liye hai jahan pehle se koi website/app chal rahi ho. Maqsad **dusri, alag website** add karna hai—existing website, uska domain, database, Nginx config, PHP version, firewall aur SSH setup ko bina wajah badle.

Commands SSH ke **existing sudo user** ke terminal mein chalayein. Yeh commands Ubuntu 24.04 + Nginx + MySQL assume karti hain. Agar VPS Apache, CloudPanel, HestiaCP ya kisi aur control panel se chalta hai, to Nginx configuration aur commands ko copy-paste na karein; panel se nayi website banayein aur web root is project ke `public/` folder par point karein.

## 1. Nayi site ke liye alag subdomain rakhein

Misal: agar existing site `example.com` par hai, nayi attendance site ke liye `attendance.example.com` use karein. Existing domain ki DNS entries ya website config ko badalne ki zaroorat nahi honi chahiye.

1. Hostinger VPS ka **existing** public IP note karein.
2. Domain DNS mein naya `A` record add karein: `attendance` → VPS ka public IPv4.
3. DNS parivartan publish hone tak intezar karein. Agar VPS IPv6 configure nahi hai, naye subdomain ke liye `AAAA` record add na karein.
4. Hostinger panel mein dekh lein ki isi subdomain ko pehle se koi aur site use nahi kar rahi.

`attendance.example.com` ko apne asli subdomain se replace karein. Windows se VPS par pehle se use hone wale SSH login se connect karein:

```powershell
ssh YOUR_EXISTING_USER@YOUR_SERVER_IP
```

## 2. Existing server ko check karein; uski configuration na hatayein

```bash
lsb_release -a
nginx -v
php -v
php -m
composer --version
node --version
ls -1 /run/php/
sudo nginx -T
```

`nginx -T` se current virtual hosts aur unke `server_name` check karein. Existing setup ka backup ya Hostinger VPS snapshot bana kar confirm karein ki usse restore kar sakte hain.

**Firewall mein reset/enable na karein, SSH config na badlein, Nginx ka default ya purana site config na delete karein, aur doosri site ke files par `chown`, `chmod` ya deployment commands na chalayein.** Existing site chal rahi hai to uski Nginx, PHP-FPM, MySQL aur firewall settings use karein. Agar project ke liye PHP extensions ki kami ho, sirf woh extensions add karein; existing PHP-FPM socket/version ko badalne se pehle administrator se check karein.

Project requirements: PHP `8.3+`, Composer 2, MySQL/MariaDB aur Node.js `22.13+` for build. `php -m` mein `bcmath`, `curl`, `fileinfo`, `intl`, `mbstring`, `openssl`, `PDO`, `pdo_mysql`, `xml` aur `zip` check karein. Pehle se compatible Node, Composer, PHP aur MySQL installed hain to unhein dobara install na karein.

## 3. Sirf project-specific database banayein

Agar Hostinger/control panel database banane ka option deta hai, usse **naya database aur naya database user** banayein; credentials note karke aage `.env` mein daalein. Existing site ka database/user reuse na karein.

Ya, agar server par MySQL admin access available hai:

```bash
sudo mysql
```

MySQL prompt mein ek unique, strong password dekar naya database aur user banayein:

```sql
CREATE DATABASE timetracker CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'timetracker'@'localhost' IDENTIFIED BY 'YOUR_UNIQUE_DATABASE_PASSWORD';
GRANT ALL PRIVILEGES ON timetracker.* TO 'timetracker'@'localhost';
EXIT;
```

MySQL `root` user app ke `.env` mein na daalein. Database ko public internet par expose na karein.

## 4. Is project ki files alag folder mein rakhein

Is repository ke **latest aur intended changes** apne Git server par push karein. `.env`, secrets, `vendor`, `node_modules` aur local database ko Git mein upload na karein. Private repository ke liye VPS ki SSH deploy key use karein; token ko clone URL mein na daalein. Local changes push kiye bina `git clone` karne par VPS par purana code aayega.

```bash
sudo mkdir -p /var/www/timetracker
sudo chown YOUR_EXISTING_USER:www-data /var/www/timetracker
cd /var/www/timetracker
git clone YOUR_GIT_REPOSITORY_URL .
```

`YOUR_EXISTING_USER` ko apne SSH login name se aur repository URL ko apne Git URL se replace karein. Folder mein pehle se files hon to unhein overwrite/delete karne se pehle check karein.

Git istemal nahi kar rahe to WinSCP/SFTP se **sirf isi naye project folder** mein app files upload karein; `.env` upload na karein. `vendor/` aur `node_modules/` upload karna zaroori nahi—dependencies server par install hongi.

## 5. Nayi site ka `.env` banayein

```bash
cd /var/www/timetracker
cp .env.example .env
nano .env
```

Naye subdomain aur **isi naye project** ke database credentials set karein:

```dotenv
APP_NAME="Time Tracker"
APP_ENV=production
APP_KEY=
APP_DEBUG=false
APP_URL=https://attendance.example.com
LOG_LEVEL=warning

DB_CONNECTION=mysql
DB_HOST=localhost
DB_PORT=3306
DB_DATABASE=timetracker
DB_USERNAME=timetracker
DB_PASSWORD='YOUR_UNIQUE_DATABASE_PASSWORD'

SESSION_DRIVER=file
SESSION_SECURE_COOKIE=true
CACHE_STORE=file
QUEUE_CONNECTION=database

ADMIN_EMAIL=admin@example.com
ADMIN_PASSWORD='YOUR_UNIQUE_ADMIN_PASSWORD'
```

`APP_URL`, database fields aur admin details ko apne values se replace karein. Admin password production seeding se pehle set karein, unique aur strong rakhein, aur kisi ko chat ya screenshot mein na bhejein. Is project ke migrations mein session table nahi hai; `SESSION_DRIVER=file` aur `CACHE_STORE=file` rakhein. `DB_HOST=localhost` local MySQL socket use karta hai aur upar banaye MySQL `'timetracker'@'localhost'` user se match karta hai.

## 6. PHP aur asset dependencies set up karein

Pehle check karein ki server ke default `php` command se project supported version chala raha hai:

```bash
cd /var/www/timetracker
php -v
composer --version
node --version
npm --version
```

Agar PHP version `8.3` se kam ho, Node.js `22.13` se purana ho ya koi required PHP extension missing ho, shared runtime ko andhadhundh upgrade na karein. Existing app ka runtime pehle check karein. Zaroorat par sirf naye project ke liye separate PHP-FPM version ya per-user Node.js version administrator ke saath set up karein.

Dependencies install karke production frontend build karein:

```bash
cd /var/www/timetracker
composer install --no-dev --prefer-dist --optimize-autoloader --no-interaction
npm ci --include=dev
npm run build
```

Build ke baad `public/build/manifest.json` hona chahiye. Nayi site par Vite dev server na chalayein.

## 7. App key, database migration aur admin banayein

```bash
cd /var/www/timetracker
php artisan key:generate
php artisan migrate --force
php artisan db:seed --force
```

`key:generate` sirf pehli deployment mein, is naye `.env` ke liye ek baar chalayein. Is project ke seeder ne diye hue `ADMIN_EMAIL` aur `ADMIN_PASSWORD` se admin banaya hoga. Seeder chalne ke baad `.env` se `ADMIN_PASSWORD` wali line hata dein; `APP_KEY` ko save karke surakshit backup mein rakhein. Agar admin email pehle se database mein maujood hai, seeder uska password update nahi karta.

Production par **kabhi** `php artisan migrate:fresh`, `migrate:reset` ya `db:wipe` na chalayein; in commands se data delete ho sakta hai. `storage:link` bhi na chalayein: is app ke vault uploads private hain.

## 8. Sirf naye project ki permissions set karein

`YOUR_EXISTING_USER` ko apne SSH login name se replace karein:

```bash
cd /var/www/timetracker
sudo chown -R YOUR_EXISTING_USER:www-data /var/www/timetracker
sudo find /var/www/timetracker -type d -exec chmod 2750 {} \;
sudo find /var/www/timetracker -type f -exec chmod 640 {} \;
sudo chmod -R ug+rwX /var/www/timetracker/storage /var/www/timetracker/bootstrap/cache
sudo find /var/www/timetracker/storage /var/www/timetracker/bootstrap/cache -type d -exec chmod 2770 {} \;
sudo find /var/www/timetracker/storage /var/www/timetracker/bootstrap/cache -type f -exec chmod 660 {} \;
```

Yeh commands sirf `/var/www/timetracker` ko target karti hain. Doosre project ya poore `/var/www` par recursive permission command na chalayein. Uploaded file limit app mein `20 MB` hai; Nginx aur PHP-FPM ki upload limits isse match honi chahiye.

## 9. Nginx mein sirf naya subdomain add karein

Yeh hissa tabhi use karein jab server pehle se **Nginx** use kar raha ho aur aapko virtual host add karne ka access ho. Control panel-managed VPS par pehle panel ke zariye naya subdomain/site add karein. Pehle current Nginx config mein confirm karein ki naya domain kisi existing `server_name` se conflict nahi karta.

PHP-FPM socket dekhein:

```bash
ls -1 /run/php/
```

`php8.3-fpm.sock` ko neeche apne maujooda, compatible PHP-FPM socket name se replace karein. Agar Nginx/PHP-FPM setup alag hai, apne existing vhost ki PHP `fastcgi_pass` setting follow karein.

```bash
sudo nano /etc/nginx/sites-available/timetracker
```

Naye subdomain ke liye yeh nayi site configuration banayein:

```nginx
server {
    listen 80;
    server_name attendance.example.com;
    root /var/www/timetracker/public;
    index index.php;
    client_max_body_size 22m;
    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ ^/index\.php(/|$) {
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        fastcgi_param DOCUMENT_ROOT $realpath_root;
        include fastcgi_params;
    }

    location ~ \.php$ {
        return 404;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

`attendance.example.com` aur `fastcgi_pass` socket apne hisaab se badlein. Sirf naye vhost ka symlink banayein; existing sites ke symlink ko badlein ya hatayein nahi:

```bash
sudo ln -s /etc/nginx/sites-available/timetracker /etc/nginx/sites-enabled/timetracker
sudo nginx -t
```

`nginx -t` **successful** hone ke baad hi Nginx reload karein:

```bash
sudo systemctl reload nginx
```

`sites-enabled/timetracker` pehle se maujood ho to dobara symlink banane se pehle uska target inspect karein. `sites-enabled/default` ya kisi existing site config ko remove na karein. Config test fail ho to reload na karein; naye config ko theek karein, aur agar zaroorat ho to sirf naya symlink remove karein.

20 MB upload support ke liye maujooda PHP-FPM config ki `upload_max_filesize` ko `20M` aur `post_max_size` ko `22M` karein. Agar FPM config badli ho to **sirf us PHP version** ka FPM reload karein jise nayi site use karti hai; doosri site ka PHP version ya FPM config bina jaanch ke na badlein.

## 10. Naye subdomain par HTTPS enable karein

Pehle verify karein ki naya subdomain VPS IP par resolve ho raha hai, port `80`/`443` ki **existing** Hostinger aur server firewall policy traffic allow karti hai, aur naya Nginx virtual host load hua hai. Firewall reset ya `ufw enable` na karein.

Agar VPS mein Certbot aur Nginx plugin pehle se installed hain:

```bash
sudo certbot --nginx -d attendance.example.com
sudo certbot renew --dry-run
```

Agar nahi hain, install karne se pehle VPS par existing SSL/certificate management panel aur doosri sites ka setup check karein. Control panel ya existing Certbot configuration ko bypass karke certificate renewals manage na karein.

`.env` mein HTTPS `APP_URL` rehna chahiye. Sign-in ke baad `https://attendance.example.com` par test karein.

## 11. App verify karein

```bash
cd /var/www/timetracker
php artisan optimize
php artisan about
php artisan migrate:status
curl -I https://attendance.example.com/up
```

Naye subdomain par admin email/password se sign in karke dashboard, attendance aur asset loading test karein. Pehle se chal rahi site ka URL bhi khol kar verify karein ki us par koi asar nahi pada.

Employees ke `allowed_ips` khaali hone par app IP restriction nahi lagati; IP allow-list set karne se pehle sahi client IP verify karein. Nginx reverse proxy ya VPS panel ke peeche real client IP handling ko alag se check karein.

## 12. Timezone aur automatic scheduler

Project ke `config/app.php` mein default timezone `UTC` hai. Agar shifts India local time mein chalti hain, deployment se pehle code mein `'timezone' => 'Asia/Kolkata',` set karke apne Git branch mein commit karein; isse dashboard, attendance aur scheduler ka time ek hi timezone follow karega. Sirf server ka timezone badalna kaafi nahi.

Project ka scheduler roz `00:30` par bhule hue open timers close karta hai aur `00:45` par absent mark karta hai. Pehle:

```bash
cd /var/www/timetracker
php artisan schedule:list
```

Apne **existing SSH/deploy user** ka crontab kholein:

```bash
crontab -e
```

Maujooda crontab entries ko barkarar rakhte hue yeh **ek** line add karein:

```cron
* * * * * cd /var/www/timetracker && /usr/bin/php artisan schedule:run >> /dev/null 2>&1
```

Agar CLI PHP `/usr/bin/php` par nahi hai, `command -v php` se sahi path dekhein. `crontab` ko khali file se overwrite na karein. Is app ko abhi alag queue worker ki zaroorat nahi hai.

## 13. Existing site ko bachate hue updates aur backup

Deployment se pehle naye app ka MySQL database aur private vault uploads ka backup lein. Production files mein `.env` aur `storage/app/private` dono sensitive hain; backup ko secure/encrypt karke VPS ke bahar rakhein aur restore test karein. Existing site ke database ya uploads ko is project ke commands se touch na karein.

Jab naye version ko update karna ho, latest intended changes pehle Git server par push karein, phir **sirf is project folder mein**:

```bash
cd /var/www/timetracker
php artisan down
git pull --ff-only
composer install --no-dev --prefer-dist --optimize-autoloader --no-interaction
npm ci --include=dev
npm run build
php artisan migrate --force
php artisan optimize:clear
php artisan optimize
php artisan up
```

Agar koi command fail ho, wajah theek hone tak maintenance mode se bahar na niklein. Database backup ke bina destructive migration ya `migrate:fresh` na chalayein. Purane project par `php artisan down`, `git pull`, Composer, npm ya migration commands na chalayein.

Useful checks:

```bash
sudo tail -n 100 /var/www/timetracker/storage/logs/laravel.log
sudo tail -n 100 /var/log/nginx/error.log
sudo nginx -t
sudo systemctl status nginx --no-pager
```

## References

- [Laravel 13 deployment](https://laravel.com/docs/13.x/deployment)
- [Laravel 13 task scheduling](https://laravel.com/docs/13.x/scheduling)
- [Ubuntu Server documentation](https://documentation.ubuntu.com/server/)
- [Certbot Nginx instructions](https://certbot.eff.org/instructions?ws=nginx&os=snap)
- [Node.js releases](https://nodejs.org/en/about/previous-releases)
