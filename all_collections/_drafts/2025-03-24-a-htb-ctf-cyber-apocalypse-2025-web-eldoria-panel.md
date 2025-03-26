---
layout: post
title: 'Hack The Box CTF 2025: Tales from Eldoria - Web - Eldoria Panel'
created-at: 2025-03-24
categories: ["Hack The Box", "WriteUp", "Cybersecurity", "Tales from Eldoria", "CTF"]
---

|Category|Name|Difficulty|Status|
|:---:|:---:|:---:|:---:|
|Web|Eldoria Panel|Medium|Not Completed ❌|

> A development instance of a panel related to the Eldoria simulation was found. Try to infiltrate it to reveal Malakar's secrets.

## Analysis

I started this one by directly downloading the source code. When first accessing the challenge, we are presented with a login page:

![Login page](/docs/assets/2025-03/htb-ctf-web-eldoria-panel-1.png)

It has also the option to create a new account.

When checking if I can create an account with the username `admin` I get back a message saying that the username is already taken:

![Admin account creation](/docs/assets/2025-03/htb-ctf-web-eldoria-panel-2.png)

Meaning, we probably need to access the admin account to acquire the flag.

Continuing after creating an account, we are redirected to a dashboard:

![Dashboard](/docs/assets/2025-03/htb-ctf-web-eldoria-panel-3.png)

Here we have a couple of options. There is a "Dragon's heart API key" and a "Hero Status" field. We can type a new message and click "Enchant" to update our hero status.

The options on the "Artifacts" section don't do anything.

Under the "Available Quests" section, the "+ New Quest" doesn't do anything. By clicking on "View Quest" opens a new page describing the quest:

![Quest Details](/docs/assets/2025-03/htb-ctf-web-eldoria-panel-4.png)

Nothing much can be done here.

Going back to the dashboard, on the top of the screen there is a "Claim Quest" option, where you are sent to the following page:

![Claim Quest](/docs/assets/2025-03/htb-ctf-web-eldoria-panel-5.png)

Here you can specify a quest id, a guild URL and a the number of companions. 

---

Going through the source code of the pages, I saw that there is a admin page under `/admin` and some admin specific endpoints as well.

```js
GET /api/admin/appSettings
POST /api/admin/appSettings
POST /api/admin/cleanDatabase
POST /api/admin/updateAnnouncement
```

They are all protected by a `$adminApiKeyMiddleware` function that checks for a admin session or a `X-API-Key` from an admin account.

The `/admin` page has a check that not only looks for a admin session, it also checks if the `adminCookie` is set to `true`.

```php
...
function isAdmin(Request $request) {
    return (isset($_SESSION['user'])
        && $_SESSION['user']['is_admin'] == 1
        && isset($_COOKIE['adminCookie'])
        && $_COOKIE['adminCookie'] === 'true');
}
...
```

There is also an `/bot/run_bot.py` file that simulates the behavior of a admin account. This bot is triggered by the `/api/claimQuest` API call, and it looks like the purpose is to simulate someone manually reviewing the guild website by accessing the provided URL.

```python
...
driver = webdriver.Chrome(options=chrome_options)

try:
    driver.get("http://127.0.0.1:9000")

    username_field = driver.find_element(By.ID, "username")
    password_field = driver.find_element(By.ID, "password")

    username_field.send_keys(admin_username)
    password_field.send_keys(admin_password)

    submit_button = driver.find_element(By.ID, "submitBtn")
    submit_button.click()

    driver.get(quest_url) # The provided guild URL

    time.sleep(5)

except Exception as e:
    print(f"Error during automated login and navigation: {e}", file=sys.stderr)
    sys.exit(1)
...
```

This gives us the opportunity to steal the admin session cookie. But how to acquire the flag?

I accessed the docker container and saw that there is a `sqlite` database file. I downloaded it back to my machine, and uppon analysis, it has the following data:

- A `app_settings` table that stores the running application settings (duh!);
- A `config` table that is used to store the admin announcement;
- A `quests` table that store the quest data that is show on the dashboard;
- A `users` table - that stores users.

Looking at the `app_settings` table, it looks like the path that the templates are retrieved from when rendered is stored here.

```sql
sqlite> SELECT * FROM app_settings;

template_path   |   /app/src/../templates
database_driver |   sqlite
database_name   |   /app/src/../data/database.sqlite
```

And the `users` table just confirmed my suspicion that there is an admin account.

```sql
sqlite> SELECT * FROM users;
1|admin|<random passwd>|1|817d34478052551f65cb1296841d8e19|Ready for adventure!                                          
2|hero|heropass|0|544dc162865a2a0fe692e8569700d5ed|Ready for adventure!
```

So, if I have access to the admin account, I can maybe use the `POST /api/admin/appSettings` request to replace the local templates path with a remote URL pointing to a PHP that will spawn a Shell for me.

Double checking the `render` method, it use both [file_exists](https://www.php.net/manual/en/function.file-exists.php) and [file_get_contents](https://www.php.net/manual/en/function.file-get-contents.php). Both methods that support retrieving data from a URL.

```php
function render($filePath) {
    if (!file_exists($filePath)) {
        return "Error: File not found.";
    }
    $phpCode = file_get_contents($filePath);
    ob_start();
    eval("?>" . $phpCode);
    return ob_get_clean();
}
```

## Failing at it

I did some more research on XSS, and on this specific case I cannot steal the session cooking, since it has the `httpOnly` flag enabled. Meaning that JS cannot access the cookie itself.

The target would then have to be the admin's API Key, since that can also be used to authenticate as the admin account.

```php
...
# Used by normal requests
$apiKeyMiddleware = function (Request $request, $handler) use ($app) {
	if (!isset($_SESSION['user'])) {
		$apiKey = $request->getHeaderLine('X-API-Key');
		if ($apiKey) {
			$pdo = $app->getContainer()->get('db');
			$stmt = $pdo->prepare("SELECT * FROM users WHERE api_key = ?");
			$stmt->execute([$apiKey]);
			$user = $stmt->fetch(PDO::FETCH_ASSOC);
			if ($user) {
				$_SESSION['user'] = [
					'id'              => $user['id'],
					'username'        => $user['username'],
					'is_admin'        => $user['is_admin'],
					'api_key'         => $user['api_key'],
					'level'           => 1,
					'rank'            => 'NOVICE',
					'magicPower'      => 50,
					'questsCompleted' => 0,
					'artifacts'       => ["Ancient Scroll of Wisdom", "Dragon's Heart Shard"]
				];
			}
		}
	}
	return $handler->handle($request);
};

# Used by admin specific requests
$adminApiKeyMiddleware = function (Request $request, $handler) use ($app) {
	if (!isset($_SESSION['user'])) {
		$apiKey = $request->getHeaderLine('X-API-Key');
		if ($apiKey) {
			$pdo = $app->getContainer()->get('db');
			$stmt = $pdo->prepare("SELECT * FROM users WHERE api_key = ?");
			$stmt->execute([$apiKey]);
			$user = $stmt->fetch(PDO::FETCH_ASSOC);
			if ($user && $user['is_admin'] === 1) {
				$_SESSION['user'] = [
					'id'              => $user['id'],
					'username'        => $user['username'],
					'is_admin'        => $user['is_admin'],
					'api_key'         => $user['api_key'],
					'level'           => 1,
					'rank'            => 'NOVICE',
					'magicPower'      => 50,
					'questsCompleted' => 0,
					'artifacts'       => ["Ancient Scroll of Wisdom", "Dragon's Heart Shard"]
				];
			}
		}
	}
	return $handler->handle($request);
};
...
```

I couldn't find a way steal that information though. I would need find a way to either redirect the `/api/user` result or the result of the `/dashboard` access when the bot accesses my website. But due to the different domains, I wasn't able to authenticate on an `iframe` or through `fetch`.

I also tried to figure a way to directly modify the template settings POSTing to the route on click, but no success at that either.