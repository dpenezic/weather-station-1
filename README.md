# From 24th March 2016 these instructions are superseded...
by the guides and tutorials at [https://github.com/raspberrypilearning/weather\_station\_guide](https://github.com/raspberrypilearning/weather_station_guide) (published at [www.raspberrypi.org/weather-station](https://www.raspberrypi.org/weather-station/))

This repo is for code and technical stuff.

----------


Weather Station
==============

Data logging code for the Raspberry Pi Weather Station HAT

## Instructions to deploy

1. Start with a fresh install of Raspbian. Boot up as per usual and expand the filesystem to fill the SD card.
1. Enable I²C. Enter the following command after logging into your Pi:

  `sudo raspi-config`
  
  Select `Advanced Options` and press `Enter`
  
  Select `I2C` and press `Enter`
  
  *Note: If you do not see the `I2C` option listed it means you have an old image of Raspbian. Please visit the [downloads](http://www.raspberrypi.org/downloads/) page and update your SD card to the latest version.*
  
  Would you like the ARM I2C interface to be enabled? `Yes` > `Enter`
  
  The ARM I2C interface is enabled `Ok` > `Enter`
  
  Would you like the I2C kernel module to be loaded by default? `Yes` > `Enter`
  
  I2C kernel module will now be loaded by default `Ok` > `Enter`
  
  Select `Finish` from the main menu and press `Enter`
  
  Would you like to reboot now? `Yes` > `Enter`

3. Log back in and configure the required device tree overlays. Enter the following command:

  `sudo nano /boot/config.txt`
  
  Add the following lines to the bottom of the file:
  
  ```
  dtoverlay=w1-gpio
  dtoverlay=pcf8523-rtc
  ```
  
  Press `Ctrl - O` then `Enter` to save and `Ctrl - X` to quit nano.

  Now set the required modules to load automatically on boot.

  `sudo nano /etc/modules`
  
  Add the following lines to the bottom of the file:
  
  ```
  i2c-dev
  w1-therm
  ```
  
  Press `Ctrl - O` then `Enter` to save and `Ctrl - X` to quit nano.
  
4. Ensure that a CR/BR1225 3 volt coin cell battery has been inserted into the cage. Positive `+` side facing up.
5. Reboot for the changes to take effect.

  `sudo reboot`

6. Check that the Real Time Clock appears in `/dev`
  
  `ls /dev/rtc*`
  
  Expected result: `/dev/rtc0`
  
7. Initialise the RTC with the correct time.

  Use the `date` command to check the current system time is correct. If correct then you can set the RTC time from the system clock with the following command:
  
  `sudo hwclock -w`
  
  If not then you can set the RTC time manually using the command below (you'll need to change the `--date` parameter, this example will set the date to the 1st of January 2014 at midnight):
  
  `sudo hwclock --set --date="2014-01-01 00:00:00" --utc`
  
  Then set the system clock from the RTC time.
  
  `sudo hwclock -s`
  
  You can passively display the time in the RTC using: `sudo hwclock -r`

8. Enable setting the system clock automatically at boot time. First edit the hwclock udev rule:

  `sudo nano /lib/udev/hwclock-set`
  
  Find the lines at the bottom that read:
  
  ```
  if [ yes = "$BADYEAR" ] ; then
      /sbin/hwclock --rtc=$dev --systz --badyear
  else
      /sbin/hwclock --rtc=$dev --systz
  fi
  ```

  Change the `--systz` options to `--hctosys` so that they read:
  
  ```
  if [ yes = "$BADYEAR" ] ; then
      /sbin/hwclock --rtc=$dev --hctosys --badyear
  else
      /sbin/hwclock --rtc=$dev --hctosys
  fi
  ```

  Press `Ctrl - O` then `Enter` to save and `Ctrl - X` to quit nano.

9. Install the necessary software packages.

  ```
  sudo apt-get update
  sudo apt-get install i2c-tools python-smbus telnet apache2 mysql-server python-mysqldb php5 libapache2-mod-php5 php5-mysql -y
  ```
  
  This will take some time. You will be prompted to create and confirm a password for the root user of the MySQL database server.

10. Create the database within MySQL.

  `mysql -u root -p`
  
  Enter the password that you chose during installation.
  
  You'll now be at the MySQL prompt `mysql>`, first create the database:
  
  `CREATE DATABASE weather;`
  
  Expected result: `Query OK, 1 row affected (0.00 sec)`
  
  Switch to that database:
  
  `USE weather;`
  
  Expected result: `Database changed`
  
  Create the table that will store all of the weather measurements:
  
  ```
  CREATE TABLE WEATHER_MEASUREMENT(
    ID BIGINT NOT NULL AUTO_INCREMENT,
    REMOTE_ID BIGINT,
    AMBIENT_TEMPERATURE DECIMAL(6,2) NOT NULL,
    GROUND_TEMPERATURE DECIMAL(6,2) NOT NULL,
    AIR_QUALITY DECIMAL(6,2) NOT NULL,
    AIR_PRESSURE DECIMAL(6,2) NOT NULL,
    HUMIDITY DECIMAL(6,2) NOT NULL,
    WIND_DIRECTION DECIMAL(6,2) NULL,
    WIND_SPEED DECIMAL(6,2) NOT NULL,
    WIND_GUST_SPEED DECIMAL(6,2) NOT NULL,
    RAINFALL DECIMAL (6,2) NOT NULL,
    CREATED TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY ( ID )
  );
  ```
  
  Expected result: `Query OK, 0 rows affected (0.05 sec)`
  
  Press `Ctrl - D` or type `exit` to quit MySQL.

11. Remove the fake hardware clock package.

  ```
  sudo update-rc.d fake-hwclock remove
  sudo apt-get remove fake-hwclock -y
  ```

12. Test that the I²C devices are online and working.

  `sudo i2cdetect -y 1`
  
  Expected output:
  
  ```
       0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
  00:          -- -- -- -- -- -- -- -- -- -- -- -- -- 
  10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
  20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
  30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
  40: 40 -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
  50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
  60: -- -- -- -- -- -- -- -- UU 69 6a -- -- -- -- -- 
  70: -- -- -- -- -- -- -- 77                         
  ```
  
  - `40` = HTU21D. Humidity and temperature sensor.
  - `77` = BMP180. Barometric pressure sensor.
  - `68` = PCF8523. Real Time Clock, it will show as `UU` because it's reserved by the driver.
  - `69` = MCP3427. Analogue to Digital Converter on main board.
  - `6a` = MCP3427. Analogue to Digital Converter on snap off AIR board (not present on prototype version).

  Note: `40`, `77` and `6a` will only show if you have connected the **AIR** board to the main board.

13. Download the data logging code.

  ```
  cd ~
  git clone https://github.com/raspberrypi/weather-station.git
  ```
  
  This will create a new folder in the home directory called `weather-station`.
14. **IMPORTANT**
  - If you have the small HAT version of the weather board you must *skip* this step.
  - If you have the massive prototype weather board (with the cloud graphic) you must *do* this step.

  ```
  cd weather-station
  git checkout prototype
  ```
15. Start the Weather Station daemon and test it.

  `sudo ~/weather-station/interrupt_daemon.py start`
  
  Expected result: `PID: 2345` (your number will be different)
  
  A continually running process is required to monitor the rain gauge and the anemometer. These are reed switch sensors and the code uses interrupt detection. These interrupts can occur at any time as opposed to the timed measurements of the other sensors. You can use the *telnet* program to test or monitor it.
  
  `telnet localhost 49501`
  
  Expected result:
  
  ```
  Trying 127.0.0.1...
  Connected to localhost.
  Escape character is '^]'.
  OK
  ```
  
  The following text commands can be used:
  
  - `RAIN`: displays rainfall in ml
  - `WIND`: displays average wind speed in kph
  - `GUST`: displays wind gust speed in kph
  - `RESET`: resets the rain gauge and anemometer interrupt counts to zero
  - `BYE`: quits
  
  Use the `BYE` command to quit.

16. Set the Weather Station daemon to automatically start at boot time.

  `sudo nano /etc/rc.local`
  
  Insert the following lines before `exit 0` at the bottom of the file:
  
  ```
  echo "Starting Weather Station daemon..."
  /home/pi/weather-station/interrupt_daemon.py start
  ```
  
17. If you wish, you can register your Weather Station with the cloud based data platform **Initial State** so that you can see your data anywhere and share it with other schools.

  [INITIAL STATE REGISTRATION INSTRUCTIONS]
  
  
18. Add your Initial State access and bucket keys to the local Initial State credentials file. This allows the code that uploads to Initial State to know what credentials to use.

  `cd ~/weather-station`
  
  `nano credentials.initialstate.template`
  
  Replace the `YOUR_ACCESS_KEY` and `YOUR_BUCKET_KEY` parameters with your Initial State `Access_Key` and `Bucket_Key` for the weather station as specified in the weather bucket settings. The double quotes `"` enclosing these values in this file are important so take care not to remove them by mistake.
  
  Press `Ctrl - O` then `Enter` to save and `Ctrl - X` to quit nano.
  
19. Rename the Initial State credentials template file to enable it.

  `mv credentials.initialstate.template credentials.initialstate`

20. Update the MYSQL credentials to the local MySQL credentials file with the password for the MySQL *root* user that you chose during installation.

  `nano credentials.mysql`
  
  The PASSWORD field is probably the only one you need to change (unless you also chose `raspberry`). The double quotes `"` enclosing the values are important so take care not to remove them by mistake.
  
  Press `Ctrl - O` then `Enter` to save and `Ctrl - X` to quit nano.

21. The main entry points for the code are `log_all_sensors.py` and `upload_to_initialstate.py`. These will be called by the [cron](http://en.wikipedia.org/wiki/Cron) scheduler to automatically take measurements. The measurements will be saved in the local MySQL database as well as uploaded to Initial State (if you registered).

  The template crontab file `crontab.save` is provided as a default. If you wish to change the measurement or upload frequency then edit this file before going onto the next step:
  
  `nano crontab.save`
  
  Press `Ctrl - O` then `Enter` to save and `Ctrl - X` to quit nano when you're done.

22. Heads up! At this point you may wish to stop and check out the Weather Station scheme of work ([here](https://github.com/raspberrypilearning/weather-station-sow)).
23. Enable cron to automatically start taking measurements, also known as *data logging mode*. 

  `crontab < crontab.save`

  Your weather station is now live and recording data at timed intervals.
  
  You can disable data logging mode at any time by removing the crontab with the command below:
  
  `crontab -r`
  
  To enable data logging mode again use the command below:
  
  `crontab < ~/weather-station/crontab.save`
  
  *Note: Do not have data logging mode enabled while you're working through the data Collection lessons in the scheme of work.*
  
24. You can manually cause a measurement to be taken at any time with the following command:

  `sudo python ~/weather-station/log_all_sensors.py`
  
  Don't worry if you see `Warning: Data truncated for column X at row 1`, this is expected.

25. You can manually trigger an upload too with the following command:

  `sudo python ~/weather-station/upload_to_initialstate.py`
  
26. You can also view the data in the database using the following commands:

  `mysql -u root -p`
  
  Enter the password. Then switch to the `weather` database:
  
  `USE weather;`
  
  Run a select query to return the contents of the `WEATHER_MEASUREMENT` table.
  
  `SELECT * FROM WEATHER_MEASUREMENT;`
  
  After a lot of measurements have been recorded it will be sensible to use the SQL *where* clause to only select records that were created after a specific date and time:
  
  `SELECT * FROM WEATHER_MEASUREMENT WHERE CREATED > '2014-01-01 12:00:00';`
  
  Press `Ctrl - D` or type `exit` to quit MySQL.

27. View your data in [Initial State](https://www.initialstate.com/).
