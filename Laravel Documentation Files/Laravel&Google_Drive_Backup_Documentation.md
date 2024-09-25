
# **Google Drive Backup Integration for Laravel Project**

This guide outlines the steps to integrate Google Drive as a backup storage solution for your Laravel project using the `spatie/laravel-backup` package.

## **Table of Contents**
- [Installation Requirements](#installation-requirements)
- [Installation Steps](#installation-steps)
- [Configuring Google Drive as a Storage Service](#configuring-google-drive-as-a-storage-service)
- [Google API Setup](#google-api-setup)
- [Connecting with Google Credentials](#connecting-with-google-credentials)
- [Final Steps and Scheduling Backups](#final-steps-and-scheduling-backups)
- [Backing Up the Entire Project and Database](#backing-up-the-entire-project-and-database)

---

## **Installation Requirements**
1. Ensure `database dumper` & `php-zip` module are installed on XAMPP / server.

---

## **Installation Steps**

### **Step 1: Install the Backup Package**
Open your terminal and run:
```bash
composer require spatie/laravel-backup
```

### **Step 2: Publish the Spatie Configuration**
```bash
php artisan vendor:publish --provider="Spatie\Backup\BackupServiceProvider"
```
This command creates a `backup.php` configuration file in `config/backup.php`.

---

## **Configuring Google Drive as a Storage Service**

### **Step 3: Install Google Drive Filesystem Package**
```bash
composer require masbug/flysystem-google-drive-ext
```

### **Step 4: Modify `config/filesystems.php`**
In `config/filesystems.php`, add the following array under `disks`:
```php
'google' => [
    'driver' => 'google',
    'clientId' => env('GOOGLE_DRIVE_CLIENT_ID'),
    'clientSecret' => env('GOOGLE_DRIVE_CLIENT_SECRET'),
    'refreshToken' => env('GOOGLE_DRIVE_REFRESH_TOKEN'),
    'folder' => env('GOOGLE_DRIVE_FOLDER'), // Root folder or specify another folder
    //'teamDriveId' => env('GOOGLE_DRIVE_TEAM_DRIVE_ID'),
],
```

### **Step 5: Add Google Storage Driver to `AppServiceProvider.php`**
In `App/Providers/AppServiceProvider.php`, add the following function below the `boot` method:
```php
private function loadGoogleStorageDriver(string $driverName = 'google')
{
    try {
        Storage::extend($driverName, function($app, $config) {
            $options = [];

            if (!empty($config['teamDriveId'] ?? null)) {
                $options['teamDriveId'] = $config['teamDriveId'];
            }

            $client = new Client();
            $client->setClientId($config['clientId']);
            $client->setClientSecret($config['clientSecret']);
            $client->refreshToken($config['refreshToken']);

            $service = new Drive($client);
            $adapter = new GoogleDriveAdapter($service, $config['folder'] ?? '/', $options);
            $driver = new Filesystem($adapter);

            return new FilesystemAdapter($driver, $adapter);
        });
    } catch(Exception $e) {
        // Handle exceptions here
    }
}
```

Inside the `boot` method, add:
```php
$this->loadGoogleStorageDriver();
```

### **Step 6: Update Backup Destination**
In `config/backup.php`, locate the `disks` array under the `destination` section. Comment out `'local'` and add `'google'`:
```php
'disks' => [
    //'local',
    'google',
],
```

---

## **Google API Setup**

### **Step 7: Create and Set Up Google Cloud Project**
1. Go to [Google Cloud Console](https://console.cloud.google.com) and create a new project.
2. From the menu (3-bar icon), go to **Library**.
3. Search for `Google Drive API` and enable it.
4. Go to **OAuth consent screen** and create your project (choose external if offline, otherwise internal).
5. Go to **Credentials** > **Create credentials** > **OAuth client ID**.
6. Set **Application Type** to "Web Application," provide a name, and add `https://developers.google.com/oauthplayground` to **Approved redirect URIs**.
7. Download or note down the `client_id` and `client_secret`.

---

## **Connecting with Google Credentials**

### **Step 8: Obtain and Configure OAuth Credentials**
1. Go to [OAuth 2.0 Playground](https://developers.google.com/oauthplayground).
2. Select **Drive API v3** and enable the first option.
3. Click on the gear icon, check **Use your own OAuth credentials**, and paste your `OAuth Client ID` & `OAuth Client Secret`.
4. Authorize APIs, then click **Exchange authorization code for tokens**.
5. Note down the **Refresh token**.

### **Step 9: Update the `.env` File**
Add the following to your `.env` file:
```env
GOOGLE_DRIVE_CLIENT_ID=your_client_id
GOOGLE_DRIVE_CLIENT_SECRET=your_client_secret
GOOGLE_DRIVE_REFRESH_TOKEN=your_refresh_token
GOOGLE_DRIVE_FOLDER=your_folder_name
```

---

## **Final Steps and Scheduling Backups**

### **Step 10: Important Folder Configuration**
If you use a custom folder name in `.env` (`GOOGLE_DRIVE_FOLDER`), ensure the folder is created on Google Drive before running any backup command. The folder name will default to your Laravel project `APP_NAME`.

**Additional Step:**  
In `config/backup.php`, within `destination`, add this entry:
```php
'root' => env('ANY_KEY_NAME', 'Your Backup Folder Name_Default'),
```
Then, set this key in `.env`:
```env
ANY_KEY_NAME=Your_Backup_Folder_Name
```
Comment out `GOOGLE_DRIVE_FOLDER` in `.env`.

### **Step 11: Schedule the Backup Command**
In `App/Console/Kernel.php`, add this to the `schedule` method:
```php
$schedule->command('backup:run', ['--only-db'])->everyMinute();
```

Finally, run the following in the terminal to clear cache and optimize:
```bash
php artisan optimize:clear
php artisan backup:run --only-db
```

---

## **Backing Up the Entire Project and Database**

If you want to back up the entire project including the database, remove `--only-db` from the scheduled method in `Kernel.php`.

---

This organized guide provides a step-by-step approach for setting up Google Drive as a backup solution for Laravel. You can directly use it for your GitHub documentation.
