Agreed, a tag works for picking the tables, so no list is needed. I’ll check which tag those tables have in tables.yml.
The tag only chooses the tables, though. The connection comes from the job’s env file, and the prod job’s source is prod MyTTB. So the IVV E&L still needs its own job with the IVV source settings. It would be the same loader with --select tag:<ivv tag>, reading from IVV and writing to OABLMSDB prod.
Josh, OK to set it up that way?
