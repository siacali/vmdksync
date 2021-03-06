# vmdksync

This is a small and simple tool to apply the changes represented by a VMware
sparse snapshot file (typically named `<something>-delta.vmdk`) to a file or
block device which contains the original (pre-snapshot) data.

"Why would anyone want this?" you might well ask.  "I can merge my VMware
snapshots just fine using `vmware-cmd <cfg> removesnapshots`!"  And yes, you
can certainly do that -- if you're planning on continuing to use VMware.

However, if, like me, you would like to get your VMs *off* VMware and onto
something that doesn't rustle your jimmies, then this tool will come in
rather handy.  It allows you to create a snapshot of a running VM, transfer
the (large) base image(s) of your disk(s) onto a new VM server (running
something other than VMware), then shutdown the VM and quickly transfer the
snapshot file and re-apply it at the other end -- saving you quite a
considerable amount of downtime.  In my own VMware migration, I got the
process tuned to the point that a VM was unable to respond to pings for less
than 90 seconds.


## Requirements

This script is written in Ruby.  Any halfway-recent 1.8 version of Ruby
should work.  I haven't tested it on 1.9, but I'd be surprised if it didn't
work.

You will also need the `cstruct` gem.  If you use rubygems, you'll want to
follow [Ryan Tomayko's advice about RUBYOPT](http://gist.github.com/54177),
because I'm not going to `require 'rubygems'` for you.


## Using vmdksync

These instructions assume that you know how to setup the VM configuration on
the new server yourself, and that you're comfortable with the VMware console
command line.  It'll also help if you're running something like the version
of VMware I was (ESX 3.5) or can at least do your own translation.  Your
mileage may vary.  Void where prohibited.  Do not fold, mutilate, or
spindle.

1. Create a snapshot of the running VM.

        vmware-cmd <host>.vmx createsnapshot "migrate" "Migration snapshot" 1 0

1. Transfer the base VMDK file(s) to the new server.  I like to use `netcat`
(with `dd` and `pv` and all sorts of other guff) but you can do whatever you
like.  On the destination server, I ran something like:

        nc -l 31310 | pv -pter -B 512k -s <N>G | dd ibs=1M obs=32k of=/dev/vg0/vm-hda
        
    Whilst on the VMware server I ran:
    
        nc -w 3 dest 31310 < <host>-flat.vmdk

    You'll have to find and install your own netcat package on the VMware
    server, probably; the Googletubes shall provideth.

1. Once the initial transfer is complete, you can shutdown the VM.  I like
`vmware-cmd <host>.vmx stop soft`, myself.

1. Copy the snapshot file (named something like `<host>-000001-delta.vmdk`)
to the destination server.  Again, I use netcat, but since these files are
much smaller (or, at least, they darn well should be) you can scp them
without too much hassle if you're not quite so downtime-averse.

1. Now you can use `vmdksync` on the destination server, to bring the disk
up-to-date:

        vmdksync <host>-000001-delta.vmdk /dev/vg0/vm-hda

1. Start the VM on the destination server, and check everything's running
OK.

I'd highly recommend writing some scripts that do all the right things in
the right order for your environment, to minimise downtime and reduce the
chances of human error.


## Questions?  Comments?

Since I got rid of my VMware servers, I'm not really in a position to be
able to verify and fix tricky bug reports, but if you'd like to submit
patches, comments, or whatever, please feel free to e-mail
`theshed+vmdksync@hezmatt.org`.
