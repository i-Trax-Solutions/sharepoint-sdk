# SharePoint SDK for the Microsoft Graph API

> **Fork of [gwsn/sharepoint-sdk](https://github.com/gwsn/sharepoint-sdk)** — maintained by [i-Trax Solutions Inc.](https://www.i-trax.solutions)

SharePoint SDK to use SharePoint as file storage via the Microsoft Graph API.

For the Flysystem adapter (Symfony and Laravel) see the flysystem package: [gwsn/flysystem-sharepoint-adapter](https://github.com/gwsn/flysystem-sharepoint-adapter)

---

## Why This Fork?

The original `gwsn/sharepoint-sdk` has a critical bug in `ApiConnector::request()` that causes **binary file downloads to fail with a TypeError**.

### The Bug

The `request()` method unconditionally calls `json_decode()` on **all** HTTP responses, including binary file content (images, PDFs, thumbnails). When the binary content happens to parse as valid JSON (e.g., a file that starts with `[` or `{`), `json_decode()` returns an array instead of a string. This causes `FileService::readFile()` — which has a `: string` return type — to throw:

```
PHP Fatal error: Uncaught TypeError: GWSN\Microsoft\Drive\FileService::readFile():
Return value must be of type string, array returned
```

This commonly occurs when downloading thumbnails or any non-text file via the `/content` endpoint.

### The Fix

1. **`ApiConnector.php`**: The `request()` method now checks the `Content-Type` response header before attempting `json_decode()`. Only responses with `application/json` or `text/json` content types are decoded. Binary responses (file downloads, thumbnails, etc.) are returned as raw strings.

2. **`FileService.php`**: The `readFile()` method now includes a defensive type check — if `ApiConnector::request()` returns an array (for any reason), it is re-encoded to a JSON string to satisfy the `: string` return type contract.

---

## Installation

Install via Composer using the GitHub repository:

```json
{
    "repositories": [
        {
            "type": "vcs",
            "url": "https://github.com/i-Trax-Solutions/sharepoint-sdk.git"
        }
    ],
    "require": {
        "itrax-solutions/sharepoint-sdk": "^1.1"
    }
}
```

---

## Configuration

You need a registered Azure AD application with the appropriate Microsoft Graph API permissions.

1. Go to [Azure Portal](https://portal.azure.com)
2. Go to **Azure Active Directory** → **App registrations**
3. Click **New Registration** and follow the wizard (single tenant is typically sufficient)
4. Note down:
   - **Application (client) ID** → `$clientId`
   - **Directory (tenant) ID** → `$tenantId`
5. Go to **API permissions** → **Add a permission** → **Microsoft Graph**:
   - `Files.ReadWrite.All`
   - `Sites.ReadWrite.All`
   - `User.Read`
6. Click **Grant admin consent**
7. Go to **Certificates & secrets** → **New client secret** → the value is your `$clientSecret`
8. Determine your SharePoint site slug from the URL:
   - URL: `https://{tenant}.sharepoint.com/sites/{site-slug}/Shared%20Documents/Forms/AllItems.aspx`
   - Variable: `$sharepointSite = '{site-slug}'`

---

## Usage

### With Flysystem Adapter (Preferred)

```php
use GWSN\FlysystemSharepoint\FlysystemSharepointAdapter;
use GWSN\FlysystemSharepoint\SharepointConnector;
use League\Flysystem\Filesystem;

$tenantId = 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx';
$clientId = 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx';
$clientSecret = 'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx';
$sharepointSite = 'your-site-slug';

$connector = new SharepointConnector($tenantId, $clientId, $clientSecret, $sharepointSite);
$adapter = new FlysystemSharepointAdapter($connector, '/optional-prefix');
$flysystem = new Filesystem($adapter);
```

### Direct Service Usage

```php
use GWSN\Microsoft\Authentication\AuthenticationService;
use GWSN\Microsoft\Drive\DriveService;
use GWSN\Microsoft\Drive\FileService;
use GWSN\Microsoft\Drive\FolderService;

require dirname(dirname(__DIR__)) . '/vendor/autoload.php';

$tenantId = 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx';
$clientId = 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx';
$clientSecret = 'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx';
$sharepointSite = 'your-site-slug';

// Authenticate
$authService = new AuthenticationService();
$accessToken = $authService->getAccessToken($tenantId, $clientId, $clientSecret);
```

### Drive Operations

```php
$spDrive = new DriveService($accessToken);
$driveId = $spDrive->requestDriveId($siteId);
$spDrive->setDriveId($driveId);

// Check if a resource exists
$exists = $spDrive->checkResourceExists('/test');
$exists = $spDrive->checkResourceExists('/testDoc.docx');
```

### Folder Operations

```php
$spFolderService = new FolderService($accessToken, $driveId);

// List folder contents
$items = $spFolderService->requestFolderItems('/');
$items = $spFolderService->requestFolderItems('/subfolder');

// Check if folder exists
$spFolderService->checkFolderExists('/test');

// Create folders recursively
$spFolderService->createFolderRecursive('/path/to/folder');

// Delete folder
$spFolderService->deleteFolder('/path/to/folder');
```

### File Operations

```php
$spFileService = new FileService($accessToken, $driveId);

// Write a file
$result = $spFileService->writeFile('/test.txt', 'file content here');

// Read a file (returns string content — binary safe)
$content = $spFileService->readFile('/test.txt');

// Move a file
$result = $spFileService->moveFile('/test.txt', '/destination', 'newname.txt');

// Copy a file
$result = $spFileService->copyFile('/source.txt', '/destination', 'copy.txt');

// Delete a file
$spFileService->deleteFile('/test.txt');
```

---

## Changes from Original

| Version | Change |
|---------|--------|
| **1.1.0** | Fixed `ApiConnector::request()` — checks `Content-Type` header before `json_decode()` to prevent binary file downloads from being incorrectly decoded as JSON. Added defensive type check in `FileService::readFile()`. |

---

## Testing

```bash
composer run-script test
```

---

## Credits

- **Original package**: [gwsn/sharepoint-sdk](https://github.com/gwsn/sharepoint-sdk) by [Global Web Systems B.V.](https://www.globalwebsystems.nl)
- **Fork maintained by**: [i-Trax Solutions Inc.](https://www.i-trax.solutions)

## License

The MIT License (MIT). Please see [License File](LICENSE) for more information.
