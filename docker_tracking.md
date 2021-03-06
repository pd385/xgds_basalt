## Instructions for running xGDS in docker and simulating EVA tracks.

##### Initial Installation
1. Install Docker
   * Docker for Mac is here: https://docs.docker.com/docker-for-mac/install/
   * Docker for Windows is here: https://docs.docker.com/docker-for-windows/install/

    *System Requirements*  
    
      * Mac: OSX 10.11 or newer.
      * Windows: Windows 10.
      * Linux: Not tested, but current versions should work.
      * **Note**: We tested with the "stable" release of Docker

1. Download xGDS BASALT Docker Container and Load it
   * download https://xgds.org/downloads/xgds-basalt.tar.bz2, don't unzip it, and

1. Load container data into Docker  

	```
	docker load -i xgds-basalt.tar.bz2
	```  
	Or just load from stdin:
	
	```
	docker load < xgds-basalt.tar.bz2
	```  
	
	Once you have loaded it into Docker, it is safe to delete xgds-basalt.tar.bz2.
	
1. Create Docker data storage container/volume

   ```
   docker create -v /var -v /home/xgds --name basalt-data-store xgds-basalt /bin/true
   ```  

   *Note:* This creates a persistent docker container for the xGDS home directory and database storage.  You generally do *not* want to delete this container unless things are so messed up that you need to start over.
   
##### Running and using your Docker container 
1. Check if your Docker container is running:  

   ```
   docker ps -a
   ```

1. If basalt-container is not already in the list, run it:  

   ```
   docker run -t -d --volumes-from basalt-data-store --name basalt-container -p 80:80 -p 3306:3306 -p 7500:7500  -p 222:22  xgds-basalt
   ```

1. If it is there, but *status* shows "exited" or "created" rather than "Up..." , start it:  

   ```
   docker start basalt-container
   ```
   
1. Access xGDS server
   * http://localhost
   * username and password are both xgds

1. Log into the docker container
   * password is xgds

   ```
   ssh -p 222 xgds@localhost
   ```
   Windows with PUTTY installed (make sure it is on your PATH):
  
   ```
   putty -ssh -P 222 xgds@localhost
   ```
   
   This should open a new terminal where you are successfully ssh-ed in
   
1. Get SEXTANT DEM:
   * Download SEXTANT DEM from BASALT server (there are several, choose at least Hawaii_Lava_Flows.tif) :
https://basalt.xgds.org/data/dem
   * Copy from your computer to the data directory in the docker container:

   ```
   scp -P 222 <local-path-to-DEM> xgds@localhost:xgds_basalt/data/dem
   ```
   
   Windows with PUTTY installed (make sure it is on your PATH):
   
   ```
   pscp -P 222 Hawaii_Lava_Flows.tif xgds@localhost:xgds_basalt/data/dem
   ```

1. Start the track generator
   * Log into Docker container per step #4.  

   ```
   cd xgds_basalt/apps/basaltApp/scripts
   ```
   ```
   ./evaTrackGenerator.py -i 1 -p 10001 -t /home/xgds/xgds_basalt/apps/basaltApp/scripts/test_data/20161114A_EV2_trunc.csv
   ```
   ```
   ctrl-c to stop track generation
   ```

1. Create an EVA in xGDS and start it.
   * http://localhost/xgds_planner2/addGroupFlight
   * Uncheck EV2 and SA
   * Click Create
   * For the newly created EVA, click the green 'start' button on this page: http://localhost/xgds_planner2/manage/
     * Once started that row should turn light green.
     * In the terminal running the evaTrackGenerator you should see position data

1. View the generated track in Google Earth
   * http://localhost/xgds_map_server/feedPage/
   * Click on “Open in Google Earth” button to download the KML network link file with tracking data (note you only have to do this once; next time just open Google Earth).
   * Double-click the downloaded KML file to open it in Google Earth.
     * Expand xGDS Maps on the left sidebar
     * Check and expand Live Position tracks
     * You should see a Today folder, expand that and turn on the flight that is running
       * Note that this is rerunning that 20161114A_EV2_trunc.csv file.  If you want to work with other data you can create analogous csv files as pass them as parameters to the track generator
    
1. Stop the EVA:
   * For active EVA, click the red 'stop' button on this page: http://localhost/xgds_planner2/manage/
     * Once stopped that row should no longer be light green.
     * In the terminal running the evaTrackGenerator you should stop seeing position data

1. Restart web server (from within docker container):
   * sudo apachectl restart
     * sudo password will be xgds

1. Seeing apache log (from within docker container):
   * sudo tail -f -n 100 /var/log/apache2/error.log
     * ctrl-c to stop

1. Stop Docker container:
   * Docker containers are fairly lightweight but if you need to stop it, just:

   ```
   docker stop basalt-container
   ```

   * If you need to change the parameters the container is running with, you'll want to delete it (to save space) and run again per step #2:   

   ```
   docker rm basalt-container
   ```  
   ```
   docker run...
   ```

1. Bing Maps Key  
    We use Bing Maps for our xGDS map base layers.  If you want to enable the maps for testing traverse plans, you need to get a Bing Map API key from Microsoft:
    
    * Go to: https://www.bingmapsportal.com
    * Log in with (or create) a Microsoft Account and generate a map API key.
    * ssh into your running BASALT Docker container.
    * Edit ~/xgds_basalt/settings.py (both emacs and vi are available in the container)
    * Insert your API key between the empty quotes in the line setting the XGDS\_MAP\_SERVER\_MAP\_API\_KEY.

##### Storing xGDS code on your host system
If you are doing intensive developemnt connected to the xGDS codebase (e.g. for SEXTANT) you may want to store the xGDS source directory on your host machine's file system instead of inside the docker container. This might help to make changes and debugging easier, depending on your development environment.

Here is the procedure:

   1. ssh into your docker container (ssh -p 222 xgds@localhost)
   
   1. create a tar file of the xgds_basalt directory:  
     ```
     tar cvfz ./xgds_basalt.tgz ./xgds_basalt
     ```
      * *note:* on windows you can use 7Zip to manage tar files.
   
   1. Log out of docker and copy the tar file to your host system:
     ```
     scp -P 222 xgds@localhost:xgds_basalt.tgz .
     ```
      * Uncompress the tar file in the location of your choice.
   
   1. Stop the docker container:  
     ```
     docker stop basalt-container
     ```
   
   1. Delete it:  
     ```
     docker rm basalt-container
     ```
   
   1. Run the docker image like this:
     ```
     docker run -t -d -v <path to xgds_basalt on host>:/home/xgds/xgds_basalt --volumes-from basalt-data-store --name basalt-container -p 80:80 -p 3306:3306 -p 7500:7500  -p 222:22  xgds-basalt
     ```
     
     This will hide the xgds_basalt directory in the docker data volume and use the copy on your host system instead.  Any changes you make to the code on the host side will be reflected in the docker container.
     
     **Note:** If you do make changes to xGDS code you will need to follow the same procedure as when you update from git to prepare the new code and restart Apache.

##### Updating source code
If you already have your docker container set up but need to pull new source code, follow these steps:

1. Log into the docker container (see above)

1. Change to the main git directory
   ```
   cd xgds_basalt
   ```
 
1. Pull the latest changes from the top level repository
   ```
   git pull origin master
   ```
   
1. Update the submodules
   ```
   git submodule update --rebase
   ```
    
1. Optionally update javascript libraries (only if you are told you need to)
   ```
   ./manage.py bower update
   ```

1. Prepare any changes
   ```
   ./manage.py prep
   ```
 
1. Restart apache
   ```
   sudo apachectl restart
   ```  

##### Updating docker container
1. If you have anything you want to save in the xgds home directory in the docker container, be sure to back it up!  It will be erased as part of this process. You can scp or rsync to a different system and copy back after the new container is running. 

1. Stop current container:
   ```
   docker stop basalt-container
   ```
   
1. Delete existing containers and storage to save space:
   ```
   docker rm basalt-container
   docker rm basalt-data-store
   docker rmi xgds-basalt
   docker volume prune
   ```
   Confirm that you *do* want to prune unused volumes 
   
1. Follow instructions on how to load new container from step 3 onwards. 
