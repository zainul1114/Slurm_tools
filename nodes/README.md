Slurm node scripts
------------------

Some convenient scripts for working with nodes (or lists of nodes):

* Drain a node-list: ```sdrain node-list "Reason"```.
* Resume a node-list: ```sresume node-list```.
* Do a ```ps``` process status on a node-list, but exclude system processes: ```psnode node-list```.
* Print Slurm version on a node-list: ```sversion node-list```. Requires [ClusterShell](https://wiki.fysik.dtu.dk/niflheim/SLURM#clustershell).
* Check consistency of /etc/slurm/topology.conf with nodelist in /etc/slurm/slurm.conf: ```checktopology```
* Compute node OS and firmware updates using the ```update.sh``` script.


Usage
-----

Copy these scripts to /usr/local/bin/.
If necessary configure the variables in the script.

Example output from ```psnode```:

```
# psnode a058
Node a058:
Thu Jul 20 12:40:36 2017
NODELIST   NODES PARTITION       STATE CPUS    S:C:T MEMORY TMP_DISK WEIGHT AVAIL_FE REASON              
a058           1    xeon8*       mixed    8    2:4:1  23900    32752      1 xeon5570 none                
  PID S USER      STARTED     TIME %CPU   RSS COMMAND
 1213 S user01   20:27:07 00:00:00  0.0  1684 /bin/bash /var/spool/slurmd/job127132/slurm_script
 1490 S user01   20:27:13 00:00:00  0.0  4956 srun --mpi=pmi2 -n 1 fasttube
 1492 S user01   20:27:15 00:00:00  0.0   672 srun --mpi=pmi2 -n 1 fasttube
```

Compute node OS and firmware updates
------------------------------------

This procedure requires [ClusterShell](https://wiki.fysik.dtu.dk/niflheim/SLURM#clustershell).

Assume that you want to update OS and firmware on a specific set of nodes defined as ```<nodelist>```.
It is recommended to update entire partitions, or the entire cluster, at a time in order to avoid having inconsistent node states in the partitions.

First configure the ```update.sh``` script so that it will perform the required OS and firmware updates for your specific partitions.

Then copy the ```update.sh``` file to the compute nodes:
```
clush -bw <nodelist> --copy update.sh --dest /root/
```

On the compute nodes append this crontab entry:
```
clush -bw <nodelist> 'echo "@reboot root /bin/bash /root/update.sh" >> /etc/crontab'
```

Then set the nodes to make an automatic reboot (via Slurm)
as soon as they become idle (ASAP, see the ```scontrol``` manual page) 
and change the node state to ```DOWN``` with:
```
scontrol reboot ASAP nextstate=DOWN reason=UPDATE <nodelist>
```

You can now check nodes regularly (a few times per day) as the rolling updates proceed.
List the DOWN nodes with ```sinfo -lR```.

Check the status of the DOWN nodes.
For example, you may check the running kernel and the BMC version,
and use [NHC](https://wiki.fysik.dtu.dk/niflheim/Slurm_configuration#node-health-check):
```
clush -bw@slurmstate:down 'uname -r; nhc; dmidecode -s bios-version'
```

When some nodes have been updated and tested successfully, you could resume these nodes by:
```
scontrol update nodename=<nodes that have completed updating> state=resume
```
Resuming the node is actually accomplished at the end of the ```update.sh``` script by these lines:
```
scontrol reboot nextstate=resume `hostname -s`
```

