# alt-basic-auth

Protecting your phpMyAdmin login page with an extra layer of security is a smart move to safeguard your database management interface from unauthorized access and attacks. Since you want this additional security layer to be compatible with your Enpass password manager, the best solution is to implement a form-based authentication system that allows Enpass to autofill credentials seamlessly.

Below, I'll guide you through setting up a custom PHP login page that protects access to phpMyAdmin. This method involves creating a simple authentication script that checks user credentials before granting access to phpMyAdmin. This solution is secure, reliable, and allows for easy integration with password managers like Enpass.

---

## **Overview**

- **Goal:** Add an extra layer of security to the phpMyAdmin login page using a custom form-based authentication system compatible with Enpass.
- **Method:** Create a custom PHP login script that authenticates users before allowing access to phpMyAdmin.
- **Benefits:**
  - Enhances security by adding an authentication layer.
  - Allows Enpass to autofill credentials.
  - Does not require modifying phpMyAdmin's core files.

---

## **Implementation Steps**

### **Prerequisites**

- **Server Environment:** Apache with PHP installed on Ubuntu 22.04.
- **phpMyAdmin Installation:** Ensure phpMyAdmin is installed and accessible.
- **SSL Certificate:** Your server should be configured to use HTTPS to secure data transmission.

---

### **Step 1: Create a Directory Alias for phpMyAdmin**

First, to avoid modifying the phpMyAdmin core files, we'll create an alias in Apache to serve phpMyAdmin from a different URL.

1. **Locate the phpMyAdmin Alias Configuration:**

   The default Apache configuration for phpMyAdmin might be located at `/etc/apache2/conf-available/phpmyadmin.conf`.

2. **Create a Custom Alias:**

   Let's create a custom alias, say `/securephpmyadmin`, that points to the phpMyAdmin installation directory.

   **Edit Apache Configuration:**

   ```bash
   sudo nano /etc/apache2/conf-available/secure-phpmyadmin.conf
   ```

   **Add the Following Configuration:**

   ```apache
   Alias /securephpmyadmin /usr/share/phpmyadmin

   <Directory /usr/share/phpmyadmin>
       Options Indexes FollowSymLinks
       DirectoryIndex index.php
       AllowOverride None

       <IfModule mod_php7.c>
           AddType application/x-httpd-php .php
           php_flag magic_quotes_gpc Off
           php_flag track_vars On
           php_flag register_globals Off
           php_admin_flag allow_url_fopen Off
           php_value include_path .
       </IfModule>

       <IfModule mod_authz_core.c>
           Require all granted
       </IfModule>
       <IfModule !mod_authz_core.c>
           Order Allow,Deny
           Allow from All
       </IfModule>
   </Directory>
   ```

   **Save and Exit:** Press `Ctrl + X`, then `Y`, and `Enter`.

3. **Enable the Configuration and Reload Apache:**

   ```bash
   sudo a2enconf secure-phpmyadmin
   sudo systemctl reload apache2
   ```

---

### **Step 2: Create the Custom Login Script**

We'll create a `login.php` script that will authenticate the user before granting access to phpMyAdmin.

1. **Create a Directory to Hold the Authentication Scripts:**

   ```bash
   sudo mkdir /var/www/auth
   sudo chown www-data:www-data /var/www/auth
   ```

2. **Create the `login.php` Script:**

   ```bash
   sudo nano /var/www/auth/login.php
   ```

   **Content of `login.php`:**

   ```php
   <?php
   session_start();

   // Redirect to phpMyAdmin if already authenticated
   if (isset($_SESSION['authenticated']) && $_SESSION['authenticated'] === true) {
       header('Location: /securephpmyadmin/index.php');
       exit;
   }

   // Handle form submission
   if ($_SERVER['REQUEST_METHOD'] === 'POST') {
       $username = isset($_POST['username']) ? trim($_POST['username']) : '';
       $password = isset($_POST['password']) ? $_POST['password'] : '';

       // Replace these with your actual credentials
       $valid_username = 'yourUsername';
       $valid_password_hash = password_hash('yourPassword', PASSWORD_DEFAULT);

       if ($username === $valid_username && password_verify($password, $valid_password_hash)) {
           // Authentication successful
           $_SESSION['authenticated'] = true;
           header('Location: /securephpmyadmin/index.php');
           exit;
       } else {
           // Authentication failed
           $error_message = 'Invalid username or password.';
       }
   }
   ?>

   <!DOCTYPE html>
   <html lang="en">
   <head>
       <meta charset="UTF-8">
       <title>Login to phpMyAdmin</title>
       <style>
           /* Basic styling */
           body {
               font-family: Arial, sans-serif;
               background-color: #f2f2f2;
           }
           .login-container {
               max-width: 350px;
               margin: 100px auto;
               padding: 30px;
               background-color: #fff;
               border-radius: 8px;
               box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
           }
           .login-container h2 {
               margin-bottom: 25px;
               text-align: center;
           }
           .login-container label {
               display: block;
               margin-top: 15px;
           }
           .login-container input[type="text"],
           .login-container input[type="password"] {
               width: 100%;
               padding: 10px;
               margin-top: 8px;
               border: 1px solid #ccc;
               border-radius: 4px;
           }
           .login-container input[type="submit"] {
               width: 100%;
               padding: 12px;
               margin-top: 25px;
               background-color: #4285F4;
               color: #fff;
               border: none;
               border-radius: 4px;
               cursor: pointer;
           }
           .login-container .error-message {
               color: #d9534f;
               text-align: center;
               margin-top: 15px;
           }
       </style>
   </head>
   <body>
       <div class="login-container">
           <h2>Login to phpMyAdmin</h2>
           <?php
           if (isset($error_message)) {
               echo '<div class="error-message">' . htmlspecialchars($error_message) . '</div>';
           }
           ?>
           <form action="login.php" method="post">
               <label for="username">Username:</label>
               <input type="text" id="username" name="username" required autofocus>

               <label for="password">Password:</label>
               <input type="password" id="password" name="password" required>

               <input type="submit" value="Login">
           </form>
       </div>
   </body>
   </html>
   ```

   **Instructions:**

   - **Replace `'yourUsername'` and `'yourPassword'`** with your desired username and password.
   - The password is hashed using `password_hash` for security.

3. **Secure the Script:**

   - Set the appropriate permissions:

     ```bash
     sudo chown www-data:www-data /var/www/auth/login.php
     sudo chmod 640 /var/www/auth/login.php
     ```

---

### **Step 3: Restrict Access to phpMyAdmin**

We need to prevent direct access to phpMyAdmin without going through our custom login page.

1. **Create an `.htaccess` File in the phpMyAdmin Directory:**

   ```bash
   sudo nano /usr/share/phpmyadmin/.htaccess
   ```

   **Content of `.htaccess`:**

   ```apache
   <IfModule mod_rewrite.c>
       RewriteEngine On
       RewriteCond %{REQUEST_URI} !^/securephpmyadmin/logout\.php$
       RewriteCond %{REQUEST_URI} !^/securephpmyadmin/error\.php$
       RewriteCond %{REQUEST_URI} !^/securephpmyadmin/login\.php$
       RewriteCond %{REQUEST_FILENAME} !-f
       RewriteRule ^.*$ /securephpmyadmin/login.php [R=302,L]
   </IfModule>
   ```

   **Explanation:**

   - Redirects all requests to `login.php` unless the user is accessing `logout.php`, `error.php`, or `login.php`.

2. **Ensure Apache Allows `.htaccess` Overrides:**

   **Edit Apache Configuration:**

   ```bash
   sudo nano /etc/apache2/conf-available/secure-phpmyadmin.conf
   ```

   **Modify the `<Directory>` Directive:**

   ```apache
   <Directory /usr/share/phpmyadmin>
       Options Indexes FollowSymLinks
       DirectoryIndex index.php
       AllowOverride All
       ...
   </Directory>
   ```

   **Save and Exit.**

3. **Reload Apache:**

   ```bash
   sudo systemctl reload apache2
   ```

---

### **Step 4: Implement Session Management**

We need to ensure that the user remains authenticated during their session and can log out when desired.

1. **Create a Logout Script (`logout.php`):**

   ```bash
   sudo nano /var/www/auth/logout.php
   ```

   **Content of `logout.php`:**

   ```php
   <?php
   session_start();

   // Unset all session variables
   $_SESSION = [];

   // Destroy the session
   session_destroy();

   // Redirect to login page
   header('Location: login.php');
   exit;
   ?>
   ```

   **Set Permissions:**

   ```bash
   sudo chown www-data:www-data /var/www/auth/logout.php
   sudo chmod 640 /var/www/auth/logout.php
   ```

2. **Modify phpMyAdmin's Configuration to Use Sessions:**

   **Edit `config.inc.php`:**

   ```bash
   sudo nano /etc/phpmyadmin/config.inc.php
   ```

   **Add or Modify the Following Lines:**

   ```php
   <?php
   // Other configurations...

   // Use cookie authentication
   $cfg['Servers'][$i]['auth_type'] = 'cookie';

   // Enable extra security
   $cfg['Servers'][$i]['AllowNoPassword'] = false;
   ```

   **Explanation:**

   - Ensures phpMyAdmin uses its own authentication system after the custom login.
   - The user will need to log in with their MySQL credentials after passing the custom login.

---

### **Step 5: Secure the Sessions**

To enhance security, ensure that session data is protected.

1. **Set Session Parameters in `login.php` and `logout.php`:**

   At the top of `login.php` and `logout.php`, before `session_start();`, add:

   ```php
   ini_set('session.cookie_httponly', 1);
   ini_set('session.cookie_secure', 1); // Ensure you are using HTTPS
   ```

2. **Ensure HTTPS is Enabled:**

   - Install an SSL certificate (e.g., Let's Encrypt).
   - Configure Apache to redirect HTTP to HTTPS.

   **Redirect HTTP to HTTPS:**

   **Edit Apache Configuration:**

   ```bash
   sudo nano /etc/apache2/sites-available/000-default.conf
   ```

   **Add the Following Rewrite Rules:**

   ```apache
   <VirtualHost *:80>
       ServerName yourdomain.com
       Redirect / https://yourdomain.com/
   </VirtualHost>
   ```

   **Enable SSL Module and Default SSL Site:**

   ```bash
   sudo a2enmod ssl
   sudo a2ensite default-ssl
   sudo systemctl reload apache2
   ```

---

### **Step 6: Test the Implementation**

1. **Access the Custom Login Page:**

   - Navigate to `https://yourdomain.com/securephpmyadmin/login.php`.
   - The login form should appear.

2. **Authenticate Using Enpass:**

   - Save your credentials in Enpass.
   - Ensure that Enpass can autofill the login form.

3. **Verify Redirection to phpMyAdmin:**

   - After successful login, you should be redirected to phpMyAdmin's login page.
   - Authenticate using your MySQL credentials.

4. **Test Unauthorized Access:**

   - Try accessing `https://yourdomain.com/securephpmyadmin/index.php` directly without logging in.
   - You should be redirected to the custom login page.

5. **Test Logout Functionality:**

   - Create a logout link in phpMyAdmin by modifying `login.php`:

     ```html
     <a href="logout.php">Logout</a>
     ```

   - Click the logout link and ensure you're redirected back to the login page.

---

### **Step 7: Enhance Security**

1. **Protect Against Brute-Force Attacks:**

   - Implement login attempt limits.

     **In `login.php`:**

     ```php
     // After session_start();
     if (!isset($_SESSION['login_attempts'])) {
         $_SESSION['login_attempts'] = 0;
     }

     if ($_SESSION['login_attempts'] >= 5) {
         $error_message = 'Too many failed login attempts. Please try again later.';
     }
     ```

     - Increment `$_SESSION['login_attempts']` after each failed login.
     - Reset `$_SESSION['login_attempts']` after a successful login.

2. **Implement CSRF Protection:**

   - Include a CSRF token in the login form.

     **In `login.php`:**

     ```php
     // Before the HTML output
     if (empty($_SESSION['csrf_token'])) {
         $_SESSION['csrf_token'] = bin2hex(random_bytes(32));
     }
     ```

     ```html
     <input type="hidden" name="csrf_token" value="<?php echo $_SESSION['csrf_token']; ?>">
     ```

     **In the form processing logic:**

     ```php
     if (!isset($_POST['csrf_token']) || $_POST['csrf_token'] !== $_SESSION['csrf_token']) {
         $error_message = 'Invalid CSRF token.';
     }
     ```

3. **Use Strong Passwords and Hashing:**

   - Ensure your chosen password is strong and unique.
   - Use `password_hash` with `PASSWORD_BCRYPT` or `PASSWORD_DEFAULT`.

---

### **Step 8: Configure Enpass for the Custom Login**

1. **Save Credentials in Enpass:**

   - Open Enpass and create a new login entry.
   - Enter the URL: `https://yourdomain.com/securephpmyadmin/login.php`.
   - Enter your custom username and password.

2. **Enable Autofill:**

   - Ensure that Enpass browser extension is installed and enabled.
   - When you visit the login page, Enpass should prompt you to autofill the credentials.

---

### **Alternative Solution: Using Apache's `mod_auth_form`**

If you prefer not to create custom scripts, you can use Apache's `mod_auth_form` module to provide form-based authentication at the server level.

**Note:** This method is more complex and may require additional configuration. It might not be as straightforward with Enpass.

---

## **Conclusion**

By implementing a custom login script as an additional layer of security for phpMyAdmin, you achieve the following:

- **Enhanced Security:** Prevent unauthorized access to phpMyAdmin by adding a required login step.
- **Ease of Use:** Allows Enpass to autofill credentials, making it convenient for you.
- **Maintainability:** Does not modify phpMyAdmin's core files, simplifying updates.

---

## **Security Best Practices**

- **Keep phpMyAdmin Updated:** Regularly update phpMyAdmin to the latest version.
- **Restrict Access by IP (Optional):** Limit access to trusted IP addresses using Apache's `Require ip` directive.
- **Disable Root Login:** Avoid using the MySQL root account in phpMyAdmin.
- **Monitor Access Logs:** Regularly check Apache access logs for suspicious activity.

---

## **Final Notes**

- **Backup Configuration Files:** Before making changes, back up your configuration files.
- **Testing:** Thoroughly test the login process to ensure that the security layer works as intended without interfering with phpMyAdmin's functionality.
- **Documentation:** Document your changes for future reference or for other administrators.
