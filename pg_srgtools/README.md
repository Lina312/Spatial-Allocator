# How to Run the Postgres Surrogate Tool
Last updated: 7/29/2020

Note that the Postgres Surrogate Tool currently only supports creating surrogates for regular grids. For E-Grid or census track (polygon) surrogates, please use the C/C++ and Java driver version (srgtools).

0. Prerequisites
   - Install the Postgres database server and the PostGIS extension. Make sure the Postgres server is running.
   - Create a new database for the surrogates work. This document uses a database named "surrogates" by default. The following commands can be executed from the `psql` command line, and will create a database named "surrogates", connect to that database, and add the PostGIS extension.
   ```
   CREATE DATABASE surrogates;
   \c surrogates
   CREATE EXTENSION postgis;
   ```
   - For PostGIS version 3 or newer, add the separate PostGIS raster extension.
   ```
   CREATE EXTENSION postgis_raster;
   ```

   - For the surrogate tool to run, you will need to have a database user with all privileges on the new database. The following `psql` commands create a new user named "pgsurg" and assign the appropriate privileges.
   ```
   CREATE USER pgsurg WITH PASSWORD '<password>';
   GRANT ALL PRIVILEGES ON DATABASE surrogates TO pgsurg;
   GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO pgsurg;
   ```

   - Install the Java Runtime Environment (if needed).
   - Install the tcsh package (if needed).
   - Download the Spatial Allocator package. This guide uses the installation location /opt/srgtool/.

1. Update "pg_setup.csh" (located in /opt/srgtool/pg_srgtools) for your server and then run `source pg_setup.csh`.
   ```
   setenv SA_HOME    /opt/srgtool
   setenv PGBIN      /usr/bin
   setenv PG_USER    pgsurg
   setenv DBNAME     surrogates
   setenv DBSERVER   localhost
   ```
   To avoid being asked repeatedly for the Postgres user account password, you can create a password file in your home directory. See the Postgres documentation, https://www.postgresql.org/docs/current/libpq-pgpass.html, for more details.

2. Download the archive PG_surgtool.shapefiles.quick_guide.July2019.tar.gz (785 MB) from https://drive.google.com/drive/folders/1jEWk2-fECzOg2UZmJkkWw85L9wJa7A17 and unpack it in /opt/srgtool/data/. The full unpacked archive is about 1.8 GB.
   ```
   cd /opt/srgtool/data
   tar xvf PG_surgtool.shapefiles.quick_guide.July2019.tar.gz
   ```

3. Add the output modeling projection to the Postgres database.
   ```
   cd /opt/srgtool/pg_srgtools/util
   psql -h $DBSERVER -d $DBNAME -U $PG_USER -f create_900921.sql
   ```
   This command will add a new Lambert conformal conic projection with the ID 900921 to the spatial_ref_sys table in the database.

4. Load the shapefiles into tables in the database.
   ```
   cd /opt/srgtool/pg_srgtools/util
   ./load_shapefile_reproject_multi.csh
   ```
   This script will load the county boundaries shapefile (cb_2014_us_county_500k_Poly) and the population and housing shapefile (ACS_2014_5YR_PopHousing) into the database. To load additional shapefiles, edit the script and uncomment the line `source load_shapefile.csh` for the shapefiles you're interested in.

5. Create a database table representing the modeling grid.
   ```
   cd /opt/srgtool/pg_srgtools/util
   ./generate_modeling_grid.sh
   ```
   This script creates a 12 km grid using the Lambert projection added earlier.

6. Update the settings files in /opt/srgtool/pg_srgtools/ used by the surrogate tool (if needed).
   - control_variables_pg.csv
   
   | Setting | Default value | Description |
   | - | - | - |
   | PG_SERVER | localhost | Database host |
   | PG_USER | pgsurg | Postgres username |
   | DBNAME | surrogates | Database name |
   | PGBIN | /usr/bin | Location of Postgres executables |
   | OUTPUT DIRECTORY | ./outputs/us12k_444x336 | Directory for individual surrogate files |
   | LOG FILE NAME | ./LOGS/srg_conus12k.log | Log file to store all information from running the program |

   - surrogate_generation_pg.csv: Specifies which surrogates will be created. For this guide, only the population surragate (code 100) will be generated.
   
   - surrogate_specification_pg.csv: Details how each surrogate should be created, including which data and weight shapefiles should be used, which attributes to use, and any weighting or filtering functions that should be applied.

7. Run the Postgres Surrogate Tool to generate surrogates.
   ```
   cd /opt/srgtool/pg_srgtools
   ./run_pg_srgcreate.csh
   ```
   Various messages will be displayed as the tool runs. If there are no problems encountered, the last message will be
   ```
   SUCCESS -- The Program Run Completed. See log file for details.
   ```
   and you can continue to Step 8. If there is a problem, the last message will be
   ```
   ERROR -- The Program Run Stopped. See log file for details.
   ```
   The log file location is controlled by the LOG FILE NAME setting in the control_variables_pg.csv file. If you didn't change this in Step 6, the default location is /opt/srgtool/pg_srgtools/LOGS/. If your log file shows an error like the following:
   ```
   ERROR IN RUNNING THE EXECUTABLE: SRGCREATE
   SRGCREATE_ERROR>WARNING: Environment variable: MAX_LINE_SEG, not set
   SRGCREATE_ERROR>Illegal instruction
   ```
   you can try a different version of the SRGCREATE EXECUTABLE specified in the control_variables_pg.csv file. Edit control_variables_pg.csv and find the line:
   ```
   SRGCREATE EXECUTABLE,../bin/64bits/srgcreate.exe,Location of srgcreate executable
   ```
   Change "64bits" to "32bits" like so:
   ```
   SRGCREATE EXECUTABLE,../bin/32bits/srgcreate.exe,Location of srgcreate executable
   ```
   Save your changes and re-run the run_pg_srgcreate.csh script.

8. Compare your outputs to the sample outputs using the `diffsurr` tool.
   ```
   cd /opt/srgtool/pg_srgtools
   ../bin/64bits/diffsurr.exe outputs/us12k_444x336/USA_100_NOFILL.txt 100 outputs/us12k_444x336_example/USA_100_NOFILL.txt 100 0.000001
   ```
   If the newly generated surrogates match the sample outputs, you'll see the message "The surrogate comparison was successful!"
