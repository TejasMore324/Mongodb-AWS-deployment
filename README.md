
# Deploying high available MongoDB Replica set to AWS

Deploying a highly available MongoDB replica set on AWS involves careful planning, proper configuration of AWS resources, and meticulous setup of MongoDB instances. 

Distributing instances across multiple availability zones, ensuring proper security, and backup strategies are key to maintaining a robust and reliable MongoDB deployment on AWS.



Here is a detailed step-by-step guide that integrates AWS configuration and MongoDB setup.


### AWS EC2 Setup

First, prepare the AWS EC2 instances for running MongoDB and to make sure you have your own domain name.

###  1. Launch Instances

1. Launch 3 Ubuntu Server **22.04 or 24.04 LTS instances** in the EC2 console.

2. Instance Type: 
Choose i3 for NoSQL optimized instances, or ```m4.large```,```m3.medium``` for general-purpose instances.

3. Availability Zones:

Ensure each instance is in a different availability zone for high availability.

4. Security Group: 

Create a new security group named `mongodb-cluster`.

- Allow SSH on port 22 from your IP.
- Allow port 28041, 28042, 28043, 27017 from the `mongodb-cluster` security group and your IP.

5. Labels: 
Label each instance:

    - Data - db1.example.com
    - Data - db2.example.com
    - Arbiter - arbiter1.example.com

###  2. Request and Attach Elastic IPs

1. Request 3 Elastic IPs.

2. Attach the Elastic IPs to each instance to ensure consistent public IPs.

### 3. Setup DNS Records

1. DNS Console:
 Go to your domain's DNS console.(Use route 53 or GoDaddy or other DNS Management Platform)

2. Add CNAME Records:

    - `db1.example.com` -> Public DNS hostname of db1 instance.
    - `db2.example.com` -> Public DNS hostname of db2 instance.
    - `arbiter1.example.com` -> Public DNS hostname of arbiter instance.

###  Configure Servers

###  1. Set the Hostname

1. SSH into each server and set its hostname so that when we initialize the replica set, members will be able to understand how to reach one another:

    ```bash
    sudo bash -c 'echo db1.example.com > /etc/hostname && hostname -F /etc/hostname'
    ```

2. Repeat the above command for each server with the respective hostname (`db2.example.com` and `arbiter1.example.com`).

### 2. Increase OS Limits

MongoDB needs to be able to create file descriptors when clients connect and spawn a large number of processes in order to operate effectively. 

The default file and process limits shipped with Ubuntu are not applicable for MongoDB.

1. Edit limits.conf:
    ```bash
    sudo nano /etc/security/limits.conf
    ```
    Add the following lines:
    ```plaintext
    * soft nofile 64000
    * hard nofile 64000
    * soft nproc 32000
    * hard nproc 32000
    ```

2. Create 90-nproc.conf:
    ```bash
    sudo nano /etc/security/limits.d/90-nproc.conf
    ```
    Add the following lines:
    ```plaintext
    * soft nproc 32000
    * hard nproc 32000
    ```

###  3. Disable Transparent Huge Pages (THP)

Transparent Huge Pages (THP) is a Linux memory management system that reduces the overhead of Translation Lookaside Buffer (TLB) lookups on machines with large amounts of memory by using larger memory pages.

However, database workloads often perform poorly with THP, because they tend to have sparse rather than contiguous memory access patterns. You should disable THP to ensure best performance with MongoDB.

1. Create init script:
    ```bash
    sudo nano /etc/init.d/disable-transparent-hugepages
    ```
    Paste the following script:
    ```plaintext
    #!/bin/sh
    ### BEGIN INIT INFO
    # Provides:          disable-transparent-hugepages
    # Required-Start:    $local_fs
    # Required-Stop:
    # X-Start-Before:    mongod mongodb-mms-automation-agent
    # Default-Start:     2 3 4 5
    # Default-Stop:      0 1 6
    # Short-Description: Disable Linux transparent huge pages
    # Description:       Disable Linux transparent huge pages, to improve
    #                    database performance.
    ### END INIT INFO

    case $1 in
      start)
        if [ -d /sys/kernel/mm/transparent_hugepage ]; then
          thp_path=/sys/kernel/mm/transparent_hugepage
        elif [ -d /sys/kernel/mm/redhat_transparent_hugepage ]; then
          thp_path=/sys/kernel/mm/redhat_transparent_hugepage
        else
          return 0
        fi

        echo 'never' > ${thp_path}/enabled
        echo 'never' > ${thp_path}/defrag

        unset thp_path
        ;;
    esac
    ```

2. Make the script executable:
    ```bash
    sudo chmod 755 /etc/init.d/disable-transparent-hugepages
    ```
3. Set to start on boot:
    ```bash
    sudo update-rc.d disable-transparent-hugepages defaults
    ```

### 4. Configure the File System
Linux by default will update the last access time when files are modified. 

When MongoDB performs frequent writes to the filesystem, this will create unnecessary overhead and performance degradation.

1. We can disable this feature by editing the ```fstab``` file::

    ```bash
    sudo nano /etc/fstab
    ```
    Add the `noatime` flag:
    ```plaintext
    LABEL=cloudimg-rootfs   /        ext4   defaults,noatime,discard        0 0
    ```


2. In addition, the default disk read ahead settings on EC2 are not optimized for MongoDB.

 The number of blocks to read ahead should be adjusted to approximately 32 blocks (or 16 KB) of data. 
 
 We can achieve this by adding a crontab entry that will execute when the system boots up.
  
    
    sudo crontab -e


Choose ```nano``` ,if this is your first time editing the crontab, and then append the following to the end of the file:

    @reboot /sbin/blockdev --setra 32 /dev/xvda1
    

### 5. Reboot 

 Reboot the instance

    sudo reboot
    
### 6. Repeat Steps 1 - 5

 Repeat steps **1-5** for all replica set members.

## Verify Server Configuration ##

After rebooting, you can check whether the new hostname is in effect by running:

```
hostname
```
Check that the OS limits have been increased by running:
```
ulimit -u # max number of processes

ulimit -n # max number of open file descriptors
```
The first command should output 32000, the second 64000.

Check whether the Transparent Huge Pages feature was disabled successfully by issuing the following commands:
```
cat /sys/kernel/mm/transparent_hugepage/enabled

cat /sys/kernel/mm/transparent_hugepage/defrag
```
For both commands, the correct output resembles:
```
always madvise [never]
```
Check that ```noatime``` was successfully configured:
```
cat /proc/mounts | grep noatime
```
It should print a line similar to:
```
/dev/xvda1 / ext4 rw,noatime,discard,data=ordered 0 0
```
In addition, verify that the disk read-ahead value is correct by running:
```
sudo blockdev --getra /dev/xvda1
```
It should print ```32```.

Verify the configuration for all replica set members.

###  Install  MongoDB

Run the following commands to install the latest stable 6.0 version of MongoDB:

1. Install gnupg and curl:
    ```bash
    sudo apt-get install gnupg curl
    ```

2. Import the public key:
    ```bash
    curl -fsSL https://www.mongodb.org/static/pgp/server-6.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-6.0.gpg --dearmor
    ```



3. Create list file:
    ```bash
    echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-6.0.gpg ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/6.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list
    ```

4. Update package database:
    ```bash
    sudo apt-get update
    ```

5. Install MongoDB Packages:
    ```bash
    sudo apt-get install -y mongodb-org
    ```



6. Start MongoDB service:
    ```bash
    sudo systemctl start mongod
    ```

7. Check MongoDB status:
    ```bash
    sudo systemctl status mongod
    ```
8. Verify MongoDB process:
    ```bash
    ps -aef | grep mongod
    ```
## Create keyFile

The keyFile stores the password used by each node. 

The password allows each node to authenticate to each other, allowing them replicate changes between each other.

 This password should be long and very complex. 

- We’ll use the openssl command to ensure our password is complex.

```
openssl rand -base64 741 > keyFile
```
- Create the directory where the key will be stored

```
sudo mkdir -p /opt/mongodb
```
- Copy the file to the new directory

```
sudo cp keyFile /opt/mongodb
```
- Set the ownership of the keyfile to mongodb.
```
sudo chown mongodb:mongodb /opt/mongodb/keyFile
```
- Set the appropriate file permissions.

```
sudo chmod 400 /opt/mongodb/keyFile
```
- Copy the ```KeyFile```  for all replica set members.

Now it's time to configure MongoDB to operate in replica set mode, as well as allow remote access to the server.
```
sudo nano /etc/mongod.conf
```
- Find and remove ```bindIp: 127.0.0.1```, or prefix it with a ```#``` to comment it out:
```
# network interfaces
net:
  port: 27017
#  bindIp: 127.0.0.1  # remove or comment out this line
```

- Find the commented out ```security``` section and uncomment it. Use the path of the keyFile created earlier:

```
security:
  keyFile: /opt/mongodb/keyFile
  ```
 - Find the commented out ```replication``` section and uncomment it. 
  
  Add the following below, replacing ```my-replica-set``` with a name for your replica set:
  ```
  replication:
  replSetName: my-replica-set
  ```
- Restart MongoDB to apply our changes.
```
sudo systemctl restart mongod
```

###  Configure Replica Set
Be sure you have everything setup properly in all replica set members by this point.

Connect to one of the MongoDB instances (preferably ```db1```) using SSH to initialize the replica set and declare its members. 

Note that you only have to run these commands on one of the members. 

MongoDB will synchronize the replica set configuration to all of the other members automaticall


- Create directories for each instance:
    ```bash
    mkdir -p replicaset/member
    ```



 - Start MongoDB with Replica Set Configuration on each instance:

    ```
    nohup mongod --port 28041 --bind_ip localhost,db1.example.com --replSet replica_1 --dbpath replicaset/member &

    nohup mongod --port 28042 --bind_ip localhost,db2.example.com --replSet replica_1 --dbpath replicaset/member &

    nohup mongod --port 28043 --bind_ip localhost,arbiter.example.com --replSet replica_1 --dbpath replicaset/member &
    ```

## Initialize the Replica Set

- Connect to the first server (primary):
    ```bash
    mongosh --host db1.example.com --port 28041
    ```

- Create the replica set configuration:
    ```javascript
    rsconf = {
      _id: "replica_1",
      members: [
        { _id: 0, host: "db1.example.com:28041" },
        { _id: 1, host: "db2.example.com:28042" },
      ]
    }
    ```

- Initiate the replica set:
    ```javascript
    rs.initiate(rsconf)
    ```

- Check the status:
    ```javascript
    rs.status()
    ```

- Check the current replica set configuration:
    ```javascript
    rs.conf()
    ```
## Create Admin Account

The default MongoDB configuration is wide open, meaning anyone can access the stored databases unless your network has firewall rules in place.

- Create an admin user to access the database.

  ```
   mongosh
  ```

- Select admin database.

 ```
  use admin
  ```

- Create admin account.

```
db.createUser( {
    user: "johndoe",
    pwd: "strongPassword",
    roles: [{ role: "root", db: "admin" }]
}); 
```
It's recommended to not use special characters in the password to prevent issues logging in
### Add an Arbiter


- Set default write concern:
    ```javascript
    db.adminCommand({
      setDefaultRWConcern: 1,
      defaultWriteConcern: { w: 2 }
    })
    ```
- Add the arbiter:
    ```javascript
    rs.addArb("arbiter.example.com:28043")
    ```

- If needed, remove the arbiter:
    ```javascript
    rs.remove("arbiter.example.com:28043")
    ```
- Verify Replica Set Status

Take a look at the replica set status by running:
```javascript
rs.status()
```
Inspect the members array. Look for one ```PRIMARY```, one ```SECONDARY```, and one  ```ARBITER``` member. 

All members should have a health value of ```1```.

### Connect MongoDB with Authentication

Using Command Line
```
mongosh -u johndoe -p strongPassword --authenticationDatabase admin
```
Enter password when prompted.

To properly fetch admin account info, use --authenticationDatabase admin when accessing MongoDB

### Using Connection String
```
mongodb://johndoe:strongPassword@db1.example.com,db2.example.com/dbName?authSource=admin?replicaSet=example-replica-set
```
Don't forget to change:


-`user` and `password` to your own


-`example.com` to your own domain

-`dbName` to your own database name

-`example-replica-set` to your own replica set name

## Automated Backup to AWS S3 ##


###  Create the Backup Script

 **Create a new script file:**
   - Open a terminal on your server.
   - Navigate to the `/home/ubuntu` directory:
     ```sh
     cd /home/ubuntu
     ```
   - Create the backup script file:
     ```sh
     sudo nano backup.sh
     ```

 **Copy the script into the file:**
   - Copy the following script and paste it into the `backup.sh` file:

```sh
#!/bin/sh

# Make sure to:
# 1) Name this file backup.sh and place it in /home/ubuntu
# 2) Run sudo apt-get install awscli to install the AWSCLI
# 3) Run aws configure (enter s3-authorized IAM user and specify region)
# 4) Fill in DB host + name
# 5) Create S3 bucket for the backups and fill it in below (set a lifecycle rule to expire files older than X days in the bucket)
# 6) Run chmod +x backup.sh
# 7) Test it out via ./backup.sh
# 8) Set up a daily backup at midnight via crontab -e:
#    0 0 * * * /home/ubuntu/backup.sh > /home/ubuntu/backup.log 2>&1

# DB host (secondary preferred as to avoid impacting primary performance)
HOST=db2.example.com

# DB name
DBNAME=dbName

# S3 bucket name
BUCKET=mongodb/backup

# Current time
TIME=`/bin/date +%m-%d-%Y-%T`

# Username
USERNAME=johndoe

# Password
PASSWORD=strongPassword

# Log
echo "Backing up $HOST/$DBNAME to s3://$BUCKET/ on $TIME";

S3PATH="s3://$BUCKET/"
S3BACKUP=$S3PATH$TIME.gz
S3LATEST=$S3PATH"latest".gz

# Make S3 bucket
/usr/bin/aws s3 mb $S3PATH

# Dump from MongoDB data to S3
/usr/bin/mongodump -h $HOST -d $DBNAME -p $PASSWORD -u $USERNAME --authenticationDatabase "admin" --gzip --archive | aws s3 cp - $S3BACKUP

# Copy the new backup to latest
/usr/bin/aws s3 cp $S3BACKUP $S3LATEST

# All done
echo "Backup available at https://s3.amazonaws.com/$BUCKET/$TIME.gz"

# To restore database
# aws s3 cp s3://mongodb/backup/latest.gz - | mongorestore -h db1.example.com -d dbName --archive --gzip -u johndoe -p strongPassword --authenticationDatabase admin
```


##  Install AWS CLI

- **Install AWS CLI:**
    
    Run the following commands to install AWS CLI:
     ```
     sudo apt-get install awscli -y
     ```

 **Configure AWS CLI:**
  ```
  [default] 
  aws_access_key_id = xxxxxxxxxxxxxxxxxx 
  aws_secret_access_key = xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
  Default region name [None]: YOUR_REGION
  Default output format [None]: json

```
## Fill in Database and S3 Information

- **Edit the script:**
   - Open the script for editing:
     ```sh
     sudo nano /home/ubuntu/backup.sh
     ```

- **Update the database and S3 bucket information:**
   - Replace `db2.example.com` with your MongoDB host.
   - Replace `dbName` with your MongoDB database name.
   - Replace `mongodb/backup` with your S3 bucket name.

- **Update the authentication details:**
   - Replace `johndoe` with your MongoDB ```username```.
   - Replace `strongPassword` with your MongoDB ```password```.




- **Make the script executable:**
   - Run the following command to change the script's permissions:
     ```sh
     chmod +x /home/ubuntu/backup.sh
     ```

- **Run the script manually to test it:**
   - Execute the script:
     ```sh
     ./backup.sh
     or
     bash backup.sh

     ```

## Set Up Daily Backup with Crontab

- **Open crontab:**
   
    Run `sudo nano crontab -e` to edit the crontab file.

- **Schedule the script to run daily at midnight:**
   
    Add the following line to the crontab file:
    
     ```sh
     0 0 * * * /home/ubuntu/backup.sh > /home/ubuntu/backup.log 2>&1
     ```

 If it's your first time editing crontab, you might be asked to choose an editor. 
 
 Select one and then add the line above.

