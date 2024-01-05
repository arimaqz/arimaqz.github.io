---
layout: post
title: Thick Client Domination
author: arimaqz
categories: [Reverse]
tags: [re]
---


# Introduction
In this article I'll demonstrate how to show off your own thickness to the application by tearing apart the [Damn Vulnerable Thick Client App](https://github.com/srini0x00/dvta).

## **Table of Contents**
- [Architecture](#architecture)
- [Low hanging](#low-hanging)
   - [Application signing](#application-signing)
   - [Binary protection](#binary-protection)
   - [Code obfuscation](#code-obfuscation)
- [Patching the application](#patching-the-application)
   - [Enabling configure server](#enabling-configure-server)
   - [Backdooring](#backdooring)
- [Register functionality](#register-functionality)
- [Login functionality](#login-functionality)
   - [Registry](#registry)
		- [File analysis](#file-analysis)
   - [SQLi](#sqli)
- [Expenses functionality](#expenses-functionality)
	- [Viewing another user's expenses](#viewing-another-users-expenses)
	- [Retrieving all users' expenses](#retrieving-all-users-expenses)
- [Backup to FTP](#backup-to-ftp)
   - [Source code](#source-code)
   - [Sniffing](#sniffing)
- [DLL mischief](#dll-mischief)
   - [Hijacking](#hijacking)
   - [Sideloading](#sideloading)      

---

## Architecture
First things first we should check the application's architecture. This can be done by using tools like [CFF Explorer](https://ntcore.com/):

![architecture](/assets/img/posts/2024-1-5-thick-client-domination/1.png)

So this is a .NET application, good for us! if it's not obfuscated or obfuscated using public tools, we can easily read through the source code by decompiling the application. if it was developed in C/C++, we would've had to reverse engineer it and as the saying goes:
> All source code is open source if you can read assembly.

For decompiling this .NET application I'll be using [dnSpy](https://github.com/dnSpy/dnSpy).

I won't go through the source code for now, I'll go through the application and as we proceed, I'll take the time to read the source code as needed.

## Low hanging
These are the first things you may check while dealing with thick client applications including but not limited to:
- application signing
- binary protection
    - ASLR
    - safeseh
    - DEP
    - ..
- code obfuscation.

### Application signing
To check if the application has a valid and verified signature, `sigcheck.exe` from sysinternals suite may help you:

![files](/assets/img/posts/2024-1-5-thick-client-domination/19.png)

### Binary protection
To check the various binary protections, [pesecurity](https://github.com/NetSPI/PESecurity) is at your service:

![files](/assets/img/posts/2024-1-5-thick-client-domination/20.png)

### Code obfuscation
It's really simple to check if the code is obfuscated or not by decompiling the application and see if you can read through it. And if it is, you can check deobfuscators and check whether you can make it clearer or not.

## Patching the Application
Patching an application is all about modifying it to your liking and even adding some backdoor in it.

### Enabling configure server
When you first open the application, you cannot click on 'Configure Server' button because it's disabled. To enable it we have to go through the source code and find the code where the author disabled the button and enable it.

after decompiling the application, there are some modules and one of them is the Login module. in the login module, the constructor is responsible for disabling the button.

```c#
public Login()
{
	this.InitializeComponent();
	if (this.IsBeingDebugged())
	{
		Environment.Exit(1);
	}
	if (this.isServerConfigured())
	{
		MessageBox.Show("This application is usable only after configuring the server");
		return;
	}
	this.configserver.Enabled = false;
	this.configserver.Text = "hello";
}
```

`this.configserver.Enabled = false` is the culprit here and has to be patched:

```c#
this.configserver.Enabled = true;
```

Now we can set the server IP and login into the application after compiling it again using dnSpy:

![configure server](/assets/img/posts/2024-1-5-thick-client-domination/2.png)

### Backdooring
Backdooring a legitimate application is one of the oldest tricks in the book. In this scenario I'll add some code in the login method to encode and store the credentials in a file:

```c#
string path = "c:/creds.txt";
string createText = Convert.ToBase64String(Encoding.UTF8.GetBytes(@string + ":" + string2)) + Environment.NewLine;
File.WriteAllText(path, createText);
```

And when the admin logs in with the credentials `admin:admin123`, I use the following powershell commands to decode it after storing it:

```powershell
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String("YWRtaW46YWRtaW4xMjM=")) #admin:admin123
```

All that's left is to share this modified application with your victim in your next assessment!

## Register functionality
To register I'll use the following data:

```
arima:arima
arima@test.com
```

Going through the source code again, We can see another module called Register and when the register buttion is clicked, the following method is executed:

```c#
private void btnReg_Click(object sender, EventArgs e)
{
	string username = this.txtRegUsername.Text.Trim();
	string password = this.txtRegPass.Text.Trim();
	string confirmpassword = this.txtRegCfmPass.Text.Trim();
	string email = this.txtRegEmail.Text.Trim();
	if (username == string.Empty || password == string.Empty || confirmpassword == string.Empty || email == string.Empty)
	{
		MessageBox.Show("Please enter all the fields!");
		return;
	}
	if (password != confirmpassword)
	{
		MessageBox.Show("Passwords do not match");
		return;
	}
	DBAccessClass dbaccessClass = new DBAccessClass();
	dbaccessClass.openConnection();
	if (dbaccessClass.RegisterUser(username, password, email))
	{
		this.txtRegUsername.Text = "";
		this.txtRegPass.Text = "";
		this.txtRegCfmPass.Text = "";
		this.txtRegEmail.Text = "";
		MessageBox.Show("Registration Success");
	}
	else
	{
		MessageBox.Show("Registration Failed");
	}
	dbaccessClass.closeConnection();
}
```

Nothing interesting really.
## Login functionality
Back to Login module because now we have to login into the application. This is where things get interesting.
after pressing the login button the following method is called:

```c#
private void btnLogin_Click(object sender, EventArgs e)
{
	string username = this.txtLgnUsername.Text.Trim();
	string password = this.txtLgnPass.Text.Trim();
	if (username == string.Empty || password == string.Empty)
	{
		MessageBox.Show("Please enter all the fields!");
		return;
	}
	DBAccessClass db = new DBAccessClass();
	db.openConnection();
	SqlDataReader data = db.checkLogin(username, password);
	if (!data.HasRows)
	{
		MessageBox.Show("Invalid Login");
		this.txtLgnUsername.Text = "";
		this.txtLgnPass.Text = "";
		db.closeConnection();
		return;
	}
	int isadmin = 0;
	while (data.Read())
	{
		string user = data.GetString(1);
		string pass = data.GetString(2);
		string email = data.GetString(3);
		isadmin = (int)data.GetValue(4);
		if (user != "admin")
		{
			RegistryKey registryKey = Registry.CurrentUser.CreateSubKey("dvta");
			registryKey.SetValue("username", user);
			registryKey.SetValue("password", pass);
			registryKey.SetValue("email", email);
			registryKey.SetValue("isLoggedIn", "true");
			registryKey.Close();
		}
	}
	this.txtLgnUsername.Text = "";
	this.txtLgnPass.Text = "";
	if (isadmin != 1)
	{
		base.Close();
		new Main().ShowDialog();
		Application.Exit();
		return;
	}
	base.Hide();
	new Admin().ShowDialog();
	Application.Exit();
}
```

### Registry
the following if statement comes to attention:

```c#
if (user != "admin")
{
	RegistryKey registryKey = Registry.CurrentUser.CreateSubKey("dvta");
	registryKey.SetValue("username", user);
	registryKey.SetValue("password", pass);
	registryKey.SetValue("email", email);
	registryKey.SetValue("isLoggedIn", "true");
	registryKey.Close();
}
```

So when the user is not an admin, its credentials is stored in the registry in plaintext. we can retrieve it by navigating to HKCU\dvta in registry editor or in command line:

![reg](/assets/img/posts/2024-1-5-thick-client-domination/3.png)

as plain as day! even the password is stored in plaintext and retrieved with no hindrance.

#### File analysis
How would we find out some stuff are getting written to the registry if we didn't have access to the source code? There are many ways but in this scenario I'll use good ol' process hacker:

![files](/assets/img/posts/2024-1-5-thick-client-domination/12.png)

### SQLi
To confirm whether the user exists or not, the author added the following lines:

```c#
DBAccessClass db = new DBAccessClass();
db.openConnection();
SqlDataReader data = db.checkLogin(username, password);
```

navigating to where `checkLogin` function is in `DBAccess.dll` we can see the following method:

```c#
public SqlDataReader checkLogin(string clientusername, string clientpassword)
{
	string text = string.Concat(new string[]
	{
		"SELECT * FROM users where username='",
		clientusername,
		"' and password='",
		clientpassword,
		"'"
	});
	Console.WriteLine(text);
	return new SqlCommand(text, this.conn).ExecuteReader();
}
```

Smells like SQLi to me. the final query string becomes `SELECT * FROM users where username='<username>' and password '<password>';`. So if I were to insert `admin' and 1=1;-- -` in the username field when logging in, we would probably get access to the admin account:

![sqli](/assets/img/posts/2024-1-5-thick-client-domination/4.png)
![sqli](/assets/img/posts/2024-1-5-thick-client-domination/5.png)

Ta-da! We are admin now.
More can be done using this vulnerability but I'll just leave it at that and be done with it.

## Expenses functionality
Now to check the expenses functionalities after going through login.
When adding new expenses the following method is called in `DBAccess.dll`:

```c#
public bool addExpenses(string addDt, string additem, string addprice, string addemail, string addTime)
{
	bool output = false;
	SqlCommand cmd = new SqlCommand(string.Concat(new string[]
	{
		"insert into expenses values('",
		addemail,
        "','",
		additem,
		"','",
		addprice,
		"','",
		addDt,
		"','",
		addTime,
		"')"
	}), this.conn);
	try
	{
		cmd.ExecuteNonQuery();
		output = true;
	}
	catch (Exception value)
	{
		Console.WriteLine(value);
	}
	return output;
}
```

And for viewing expenses:

```C#
public DataTable viewExpenses(string emailid)
{
	SqlDataReader rdr = new SqlCommand("select item, price, date,time from expenses where email='" + emailid + "'", this.conn).ExecuteReader();
	DataTable dataTable = new DataTable();
	dataTable.Load(rdr);
	return dataTable;
}
```

### Viewing another user's expenses
So this method retrieves expenses by filtering them using the user's email. Recall that the user's email is stored in the registry when they are not an admin. This means that we can set another user's email and retrieve their expenses without logging in into the account! In this scenario I'll change my email to rebecca's which is one of the accounts created when installing DVTA.

First I'll login using rebecca's credentials and add some expenses to view:

![rebecca expenses](/assets/img/posts/2024-1-5-thick-client-domination/8.png)

Then log out and log in to my own account which is arima and change the email in registry:

![email](/assets/img/posts/2024-1-5-thick-client-domination/6.png)

And pressing the view expenses buttion in my own account:

![view expenses](/assets/img/posts/2024-1-5-thick-client-domination/7.png)

We can see rebecca's expenses now.

### Retrieving all users' expenses
The query string used to retireve expenses is flawed:

```c#
new SqlCommand("select item, price, date,time from expenses where email='" + emailid + "'", this.conn)
```

It's vulnerable to SQLi. We can retrieve all users' expenses by changing our email to:

```
' or 1=1;-- -
```

This retireves all expenses for us!

![viewing all expenses](/assets/img/posts/2024-1-5-thick-client-domination/9.png)

## Backup to FTP
When logging in as admin we can upload a .csv file. As you know FTP is insecure and everything comes and goes plaintext which means it can be sniffed easily.

### Source code
before sniffing, there is yet another module called Admin and there exists a method which is called when the backup button is clicked:

```c#
private void btnFtp_Click(object sender, EventArgs e)
{
	this.ftptext.Text = "Please wait while uploading your data";
	new Thread(delegate()
	{
		DBAccessClass dbaccessClass = new DBAccessClass();
		dbaccessClass.openConnection();
		DataTable expensesOfAll = dbaccessClass.getExpensesOfAll();
		this.pathtodownload = Path.GetTempPath();
		this.pathtodownload += "ftp-";
		Console.WriteLine(this.pathtodownload);
		expensesOfAll.WriteToCsvFile(this.pathtodownload + "admin.csv");
		this.pathtodownload + "admin.csv";
		dbaccessClass.closeConnection();
		Admin.Upload("ftp://" + this.ftpserver, "dvta", "p@ssw0rd", this.pathtodownload + "admin.csv");
		Console.WriteLine(this.pathtodownload + "admin.csv");
	}).Start();
}
```

Here we can see the username and password of the FTP server.

### Sniffing
Suppose we didn't have access to the source code, we can still sniff the FTP traffic:

![ftp 1](/assets/img/posts/2024-1-5-thick-client-domination/10.png)

And by following stream:

![ftp 2](/assets/img/posts/2024-1-5-thick-client-domination/11.png)

## DLL mischief
These methods are one of the famous or rather infamous methods in the wild.
### Hijacking
To hijack we can use process monitor to check which DLLs are not found:

![files](/assets/img/posts/2024-1-5-thick-client-domination/13.png)

And after running the application:

![files](/assets/img/posts/2024-1-5-thick-client-domination/14.png)

There are many missing DLLs but for this scenario I'll be using the `profapi.dll` which is missing and place a malicious DLL with the same name in the application folder:

![files](/assets/img/posts/2024-1-5-thick-client-domination/15.png)

### Sideloading
I really like sideloading because you don't really break the application and it can still use the functions it wants to use because we forward it to the right location.
For this scenario I'll target `version.dll` and sideload it:

![files](/assets/img/posts/2024-1-5-thick-client-domination/16.png)

And the result:

![files](/assets/img/posts/2024-1-5-thick-client-domination/17.png)

It seems to be missing from that path! We can sideload it there with the original DLL being present in `C:/Windows/SysWOW64/`.

I developed a tool called [dll-proxy-helper](https://github.com/arimaqz/dll-proxy-helper) which retrieves all the exported functions of a given DLL and adds them to the forward statement and prints them to screen:

```
helper.exe c:/windows/syswow64/version.dll
```

```c
#pragma comment(linker, "/export:GetFileVersionInfoA=c:/windows/syswow64/version.dll.GetFileVersionInfoA")
#pragma comment(linker, "/export:GetFileVersionInfoByHandle=c:/windows/syswow64/version.dll.GetFileVersionInfoByHandle")
#pragma comment(linker, "/export:GetFileVersionInfoExA=c:/windows/syswow64/version.dll.GetFileVersionInfoExA")
#pragma comment(linker, "/export:GetFileVersionInfoExW=c:/windows/syswow64/version.dll.GetFileVersionInfoExW")
#pragma comment(linker, "/export:GetFileVersionInfoSizeA=c:/windows/syswow64/version.dll.GetFileVersionInfoSizeA")
#pragma comment(linker, "/export:GetFileVersionInfoSizeExA=c:/windows/syswow64/version.dll.GetFileVersionInfoSizeExA")
#pragma comment(linker, "/export:GetFileVersionInfoSizeExW=c:/windows/syswow64/version.dll.GetFileVersionInfoSizeExW")
#pragma comment(linker, "/export:GetFileVersionInfoSizeW=c:/windows/syswow64/version.dll.GetFileVersionInfoSizeW")
#pragma comment(linker, "/export:GetFileVersionInfoW=c:/windows/syswow64/version.dll.GetFileVersionInfoW")
#pragma comment(linker, "/export:VerFindFileA=c:/windows/syswow64/version.dll.VerFindFileA")
#pragma comment(linker, "/export:VerFindFileW=c:/windows/syswow64/version.dll.VerFindFileW")
#pragma comment(linker, "/export:VerInstallFileA=c:/windows/syswow64/version.dll.VerInstallFileA")
#pragma comment(linker, "/export:VerInstallFileW=c:/windows/syswow64/version.dll.VerInstallFileW")
#pragma comment(linker, "/export:VerLanguageNameA=c:/windows/syswow64/version.dll.VerLanguageNameA")
#pragma comment(linker, "/export:VerLanguageNameW=c:/windows/syswow64/version.dll.VerLanguageNameW")
#pragma comment(linker, "/export:VerQueryValueA=c:/windows/syswow64/version.dll.VerQueryValueA")
#pragma comment(linker, "/export:VerQueryValueW=c:/windows/syswow64/version.dll.VerQueryValueW")
``` 

This can be copied directly to the malicious DLL we are developing.
And now for the real fun:

![files](/assets/img/posts/2024-1-5-thick-client-domination/18.png)

It is indeed sideloaded.    

---

That's it for this article, Happy hacking!