Creating a New Plugin  {#tutorials_plugin}
=====================
[TOC]

This tutorial gives some background for the Corsaro plugin architecture and
describes how one could go about designing and implementing a new plugin.

Because Corsaro has been designed to make adding a new plugin as easy as
possible, there is a minimal API for writing a plugin that can process packets
and write output. In addition, some plugins will also implement the \ref
arch_corsaro_in plugin API, allowing the data that they generate to be
de-serialized and used by other programs.

An example of a plugin that implements both the \ref arch_corsaro_out and \ref
arch_corsaro_in plugin APIs is the \ref plugins_ft plugin included in the
Corsaro distribution. The \ref plugins_ft plugin extracts a _key_ comprised of
several values from the packet header and maintains psuedo-flows of packets that
match this key. The optimized binary output generated by \ref plugins_ft uses as
little storage space as possible, but at the expense of human readability. Due
to this, the \ref plugins_ft plugin also implements the \ref arch_corsaro_in API
to allow other tools to further process \ref plugins_ft data. An example of such
a tool is \ref tool_corsagg which can re-aggregate \ref plugins_ft distributions
based on sub-keys.

Because of the bidirectional nature of corsaro plugins (\ref arch_corsaro_out
and \ref arch_corsaro_in), this tutorial is split into three sections - the
first describes the general process for creating and bootstrapping a new plugin,
the second describes the API to be implemented to process packets and write
output (implementing the \ref arch_corsaro_out plugin API), while the third
section describes the API for reading existing corsaro plugin output data (the
\ref arch_corsaro_in API).

Bootstrapping a New Plugin {#tutorials_plugin_bootstrap}
==========================

Bootstrapping a new plugin generally consists of two steps:

 -# Creating boilerplate plugin code
 -# Adding references to the newly created plugin to the appropriate autotools
	 config files and to the corsaro plugin manager.

Creating Boilerplate Code {#tutorials_plugin_bootstrap_create}
-------------------------

To ready a plugin for analysis code to be added, several steps must be followed:

 -# Create \ref tutorials_plugin_bootstrap_interface
 -# Create \ref tutorials_plugin_bootstrap_impl
   -# Create \ref tutorials_plugin_bootstrap_impl_plugin
   -# Create \ref tutorials_plugin_bootstrap_impl_state
   -# Create \ref tutorials_plugin_bootstrap_impl_macro

Each corsaro plugin must have both interface (.h) and implementation (.c) files.
Currently, the convention is for these files to be placed in the
`libcorsaro/plugins/` directory to keep them separate from the base
corsaro code.

There is also a strict naming scheme which allows the plugin macros to function
correctly. That is, a plugin has both a _name_ and an _ID_. The _name_ is
used for assigning a namespace to the plugin and also for output file names,
logs, etc. The _ID_ is used to allow the plugin manager to maintain state about
each plugin currently in operation.

Each plugin is free to choose any name provided it does not conflict with
existing plugins. We suggest keeping names as brief as possible while conveying
the purpose of the plugin (remember, this is how you will identify the file(s)
each plugin has generated).  For example, we use the name 'flowtuple'
for our plugin which generates distributions of key fields (tuples) in the
packet headers.

As mentioned earlier, the plugin name provides a unique namespace for each
plugin. To this end, the source files for each plugin must follow the following
format:

~~~
corsaro_<plugin_name>.[ch]
~~~

Building on our previous example, the \ref plugins_ft code is
located in the following two files:
   
~~~
libcorsaro/plugins/corsaro_flowtuple.c
libcorsaro/plugins/corsaro_flowtuple.h
~~~

### Interface Code ###{#tutorials_plugin_bootstrap_interface}

The simplest of the two files needed for each plugin is the interface (.h) file.
A minimal plugin needs only a single line in the header file:

~~~
CORSARO_PLUGIN_GENERATE_PROTOS(<plugin_name>)
~~~

The \ref CORSARO_PLUGIN_GENERATE_PROTOS macro is located in \ref
corsaro_plugin.h and will expand to all of the required function prototypes to
satisfy the plugin API. This is another example of why the plugin name is
important - this macro assumes that the functions which implement the plugin API
are prefixed with the name of the plugin. This can be seen when we look at the
function prototypes that would be generated (by the pre-processor) for the \ref
plugins_ft plugin:

~~~{.c}
int corsaro_flowtuple_probe_filename(const char *fname);                       
int corsaro_flowtuple_probe_magic(struct corsaro_in * corsaro, corsaro_file_in_t *file); 
int corsaro_flowtuple_init_input(struct corsaro_in *corsaro);                  
int corsaro_flowtuple_init_output(struct corsaro *corsaro);                    
int corsaro_flowtuple_close_input(struct corsaro_in *corsaro);                 
int corsaro_flowtuple_close_output(struct corsaro *corsaro);                   
off_t corsaro_flowtuple_read_record(struct corsaro_in *corsaro,
                           enum corsaro_in_record_type *record_type,    
                           struct corsaro_in_record *record);           
off_t corsaro_flowtuple_read_global_data_record(struct corsaro_in *corsaro,    
                           enum corsaro_in_record_type *record_type,    
                           struct corsaro_in_record *record);           
int corsaro_flowtuple_start_interval(struct corsaro *corsaro,                  
                           struct corsaro_interval *int_start);      
int corsaro_flowtuple_end_interval(struct corsaro *corsaro,                    
                           struct corsaro_interval *int_end);          
int corsaro_flowtuple_process_packet(struct corsaro *corsaro,                  
                           struct corsaro_packet *packet);
~~~							  

These prototypes specify the names of the functions that the implementation file
must contain.

If you look at the actual header file (\ref corsaro_flowtuple.h) for the \ref
plugins_ft plugin, you will find lots of other code also - this is used for the
\ref arch_corsaro_in API and is discussed in the \ref
tutorials_plugin_corsaro_in section.

### Implementation Code ###{#tutorials_plugin_bootstrap_impl}

Moving to the implementation (.c) file, there is two data structures that we
recommend you define before implementing the API functions. These provide some
state for the plugin (because there can potentially be multiple instances of
Corsaro used simultaneously) and helper macros for accessing that state.

### corsaro_plugin Instance ###{#tutorials_plugin_bootstrap_impl_plugin}

The first structure that is needed is an instance of \ref corsaro_plugin which
describes the plugin. This will be copied by the plugin manager when an instance
of the plugin is created, so should be declared as `static`. The plugin name and
magic number are at the discretion of the plugin so long as they are unique
across plugins. The plugin ID must match that which is listed in \ref
corsaro_plugin.h (adding a new ID is described in \ref
tutorials_plugin_bootstrap_integrate section). The remainder of the fields can
be filled in using the \ref CORSARO_PLUGIN_GENERATE_PTRS macro. The following is
the \ref corsaro_plugin definition for the \ref plugins_ft plugin:

~~~{.c}
static corsaro_plugin_t corsaro_flowtuple_plugin = {
  PLUGIN_NAME,                                        /* name */
  CORSARO_PLUGIN_ID_FLOWTUPLE,                        /* id */
  CORSARO_FLOWTUPLE_MAGIC,                            /* magic */
  CORSARO_PLUGIN_GENERATE_PTRS(corsaro_flowtuple),    /* func ptrs */
  NULL,                                               /* next */
};
~~~

#### State Structure(s) ###{#tutorials_plugin_bootstrap_impl_state}

The second structure needed is a structure that will hold any per-instance state
needed by the plugin. For example, pointers to output files, data structures for
analysis, etc. This structure can be in any format, the pointer will be cast to
`void` by the plugin manager, but using a well-structured name will allow the
use of convenience macros described below for retrieving the state from the
plugin manager.  The following is the definition of the state structure for the
\ref plugins_ft plugin:

~~~{.c}
struct corsaro_flowtuple_state_t {
  /** Array of hash tables, one for each corsaro_flowtuple_class_type_t */
  khash_t(sixt) *st_hash[CORSARO_FLOWTUPLE_CLASS_MAX+1];
  /** The outfile for the plugin */
  corsaro_file_t *outfile;
};
~~~

If implementing the \ref arch_corsaro_in API, the plugin will also require a
state structure to use in \ref arch_corsaro_in mode. While these two may be the
same structure, we suggest keeping them separate to avoid confusion. The
following is the \ref arch_corsaro_in state structure for the \ref plugins_ft
plugin:

~~~{.c}
struct corsaro_flowtuple_in_state_t {
  /** The expected type of the next record in the file */
  corsaro_in_record_type_t expected_type;
  /** The number of tuples in the current class */
  int tuple_total;
  /** The number of tuples already read in the current class */
  int tuple_cnt;
};
~~~

### Helper Macros ###{#tutorials_plugin_bootstrap_impl_macro}

As mentioned earlier, there are two macros provided by the plugin manager (\ref
corsaro_plugin.h) which assist with the retrieval of plugin and state structures
from the plugin manager. These are:

 -# \ref CORSARO_PLUGIN_PLUGIN
  - Takes a pointer to the \ref corsaro state structure, and the ID of the
	plugin to retrieve. Returns a pointer to the \ref corsaro_plugin structure
	registered for that plugin ID.
 -# \ref CORSARO_PLUGIN_STATE
  - Takes a pointer to the \ref corsaro state structure, the type of the
    structure, and the plugin ID. Returns a (correctly cast) pointer to the
    state structure registered for the given plugin ID.
  - The `type` field assumes a state structure which is named
    `corsaro_<type>_state_t`
 
While these can be used alone to retrieve state and plugin structures from the
plugin manager, it might be useful to wrap them in additional macros which are
specific to the plugin being created. The core plugins included in the Corsaro
distribution all define additional macros: one to retrieve the plugin structure,
one to retrieve the Corsaro-Out state structure, and optionally one to retrieve
the Corsaro-In state structure. The \ref plugins_ft plugin defines all three of
these as follows:

~~~{.c}
/* Extends the generic plugin state convenience macro in corsaro_plugin.h */
#define STATE(corsaro)                               \
  (CORSARO_PLUGIN_STATE(corsaro, flowtuple, CORSARO_PLUGIN_ID_FLOWTUPLE))
/* Extends the generic plugin state convenience macro in corsaro_plugin.h */
#define STATE_IN(corsaro)                            \
  (CORSARO_PLUGIN_STATE(corsaro, flowtuple_in, CORSARO_PLUGIN_ID_FLOWTUPLE))
/* Extends the generic plugin plugin convenience macro in corsaro_plugin.h */
#define PLUGIN(corsaro)                              \
  (CORSARO_PLUGIN_PLUGIN(corsaro, CORSARO_PLUGIN_ID_FLOWTUPLE))
~~~

Each of these simply takes a pointer to the \ref corsaro (or \ref corsaro_in)
structure and returns the appropriate pointer.

Once these structures and macros have been defined, stub API functions defined
by the \ref CORSARO_PLUGIN_GENERATE_PROTOS macro should be created, and then the
plugin can be integrated into Corsaro.

Integrating a New Plugin {#tutorials_plugin_bootstrap_integrate}
------------------------

There are 4 files that must be updated in order to include a new corsaro
plugin:

 -# `configure.ac`
 -# `libcorsaro/plugins/Makefile.am`
 -# `libcorsaro/corsaro_plugin.c`
 -# `libcorsaro/corsaro_plugin.h`
 
 #### configure.ac ####
 
`configure.ac` contains a list of plugins that are available for compilation
into Corsaro. Each plugin that is to be made available must be declared using
the `ED_WITH_PLUGIN` macro. This macro takes four arguments, the full name of
the plugin (e.g. `corsaro_pcap`), the short name of the plugin (e.g. `pcap`),
the 'macro' name for the plugin (e.g. `PCAP`), and whether the plugin should be
enabled by default (`yes` or `no`). The \ref plugins_pcap plugin is declared as
follows:

~~~
ED_WITH_PLUGIN([corsaro_pcap],[pcap],[PCAP],[no])
~~~

Note that the order plugins are declared in this file is the _default_ order in
which they will be run. That is, a plugin declared after all others will be
given a packet to process once all other plugins have processed it.

### libcorsaro/plugins/Makefile.am ###

Using the values defined by the `ED_WITH_PLUGIN` macro declaration in the
`configure.ac` file, add the plugin to `libcorsaro/plugins/Makefile.am` as
follows:

~~~
if WITH_PLUGIN_<macro_name>
PLUGIN_SRC += <full_name>.c <full_name>.h
endif
~~~

The corresponding `Makefile.am` entry for the \ref plugins_pcap plugin
`ED_WITH_PLUGIN` example given above is:

~~~
if WITH_PLUGIN_PCAP
PLUGIN_SRC+=corsaro_pcap.c corsaro_pcap.h 
endif
~~~

#### libcorsaro/corsaro_plugin.c ####

The code that needs to be added to `libcorsaro/corsaro_plugin.c` is very
similar to that which was added to `libcorsaro/plugins/Makefile.am` in the
previous section. The format is as follows:

~~~
#ifdef WITH_PLUGIN_<macro_name>
#include "<full_name>.h"
#endif
~~~

Using the \ref plugins_pcap example again, the entry would be:

~~~
#ifdef WITH_PLUGIN_PCAP
#include "corsaro_pcap.h"
#endif
~~~

### libcorsaro/corsaro_plugin.h ###

The final change that must be made is to add a unique ID for the plugin to the
\ref corsaro_plugin_id enum in `libcorsaro/corsaro_plugin.h`. 

The ID value can be any number not taken, but the plugin manager will allocate
memory to hold plugins for every possible value up to the maximum, so we suggest
keeping this number reasonably small. The ID value **does not** affect the order
in which plugins are run, this is (currently) determined either by the order of
the `ED_WITH_PLUGIN` macros in `configure.ac` or at run-time by using \ref
corsaro_enable_plugin. The ID value defined for a plugin in this list **must**
be the same value given in the \ref corsaro_plugin structure described in the
\ref tutorials_plugin_bootstrap_impl section.

For example, the ID definition for the \ref plugins_pcap plugin is:

~~~
CORSARO_PLUGIN_ID_PCAP             = 1,
~~~

### Testing the Plugin ###

At this point, the (stub) plugin is fully integrated into Corsaro, and should
compile cleanly. Because we have altered files needed by _autoconf_, the
`autoreconf -vfi` command should be used to regenerate `configure` and
each `Makefile`. Also remember that unless the `ED_WITH_PLUGIN` macro in
`configure.ac` has a `yes` as the final argument, the plugin will need to be
explicitly enabled by passing `--with-<short_name>` to `configure` (e.g. `--with-pcap`).

To fully rebuild everything, use the following:

~~~
autoreconf -vfi
./configure [add any options needed]
make
~~~

There is also a `build_latest.sh` script included in the distribution which will
take care of these tasks, and also build a distribution tarball that can be
tested on another system.

Implementing the Corsaro-Out API   {#tutorials_plugin_corsaro_out}
================================

This section describes each of the functions that must be implemented to fully
comply with the \ref arch_corsaro_out API. Each function has some code snippets
that are almost always used. The actual function implementations will have
`corsaro_<plugin_name>_` prefixed to the names. For example, the \ref
plugins_pcap implementation of the \ref alloc_output function is called
`corsaro_pcap_alloc`.

The required functions are:
1. \ref alloc_output
2. \ref init_output
3. \ref close_output
4. \ref start_interval
5. \ref end_interval
6. \ref process_packet

alloc {#alloc_output}
-----

The _alloc_ function is called when the plugin manager needs to instantiate a
new instance of a plugin. If you followed the steps earlier in this tutorial
(see \ref tutorials_plugin_bootstrap_impl), then this function should simply
return a pointer to the static \ref corsaro_plugin structure. The plugin
manager will then make a copy for this specific instance.

For example, the \ref plugins_pcap plugin uses the following static \ref
corsaro_plugin structure definition and _alloc_ function:

~~~{.c}
static corsaro_plugin_t corsaro_pcap_plugin = {
  PLUGIN_NAME,                                 /* name */
  CORSARO_PLUGIN_ID_PCAP,                      /* id */
  CORSARO_PCAP_MAGIC,                          /* magic */
  CORSARO_PLUGIN_GENERATE_PTRS(corsaro_pcap),  /* func ptrs */
  NULL,                                        /* next */
};
~~~
~~~{.c}
corsaro_plugin_t *corsaro_pcap_alloc(corsaro_t *corsaro)
{
  return &corsaro_pcap_plugin;
}
~~~

init_output {#init_output}
-----------

The _init_output_ function is called by Corsaro to ready a plugin for use in the
\ref arch_corsaro_out mode. This event should be used to establish any state
necessary for analyzing packets. Plugins should expect the next event to be \ref
start_interval.

We will use the \ref plugins_dos plugin as an example for a simple _init_output_
function which establishes some state, and opens an output file to write data
to. The full \ref corsaro_dos_init_output function is as follows:

~~~{.c}
int corsaro_dos_init_output(corsaro_t *corsaro)
{
  struct corsaro_dos_state_t *state;
  corsaro_plugin_t *plugin = PLUGIN(corsaro);
  assert(plugin != NULL);

  if((state = malloc_zero(sizeof(struct corsaro_dos_state_t))) == NULL)
    {
      corsaro_log(__func__, corsaro, 
		"could not malloc corsaro_dos_state_t");
      goto err;
    }
  corsaro_plugin_register_state(corsaro->plugin_manager, plugin, state);

  /* open the output file */
  if((state->outfile = corsaro_io_prepare_file(corsaro, plugin->name)) == NULL)
    {
      corsaro_log(__func__, corsaro, "could not open %s output file", 
		plugin->name);
      goto err;
    }

  state->attack_hash = kh_init(av);

  return 0;

 err:
  corsaro_dos_close_output(corsaro);
  return -1;
}
~~~

Breaking it down, there are three important steps:
 -# Allocate memory for our state structure
 -# Register the state structure with the plugin manager
 -# Open the output file
 
### Allocate Memory for State ###

Because Corsaro is designed such that multiple instances of a plugin can be used
at once, per-instance state must not be stored in static structures. Instead we
create a special _state_ structure and register it with the plugin manager so
that it can be retrieved at any time it is needed.

~~~{.c}
struct corsaro_dos_state_t *state;
...
if((state = malloc_zero(sizeof(struct corsaro_dos_state_t))) == NULL)
  {
    corsaro_log(__func__, corsaro, 
	            "could not malloc corsaro_dos_state_t");
    goto err;
  }
~~~

Corsaro provides a utility function, \ref malloc_zero, which will allocate and
zero a block of memory. If the `malloc` fails, we issue a log message using \ref
corsaro_log and jump to the `err` block which simply calls the
`corsaro_dos_close_output` function described in the next section.

### Register State with Plugin Manager ###

To register the newly created state structure with the plugin manager, we
simply call the \ref corsaro_plugin_register_state function:

~~~{.c}
corsaro_plugin_t *plugin = PLUGIN(corsaro);
...
corsaro_plugin_register_state(corsaro->plugin_manager, plugin, state);
~~~

This function takes three arguments, a pointer to the plugin manager (contained
in the \ref corsaro state structure), a pointer to the plugin which is
registering the state, and a void pointer to a state structure.  We use the
`PLUGIN` macro which was described in \ref tutorials_plugin_bootstrap_impl to
retrieve a pointer to the plugin structure.

### Open Output File ###

~~~{.c}
/* open the output file */
if((state->outfile = corsaro_io_prepare_file(corsaro, plugin->name)) == NULL)
  {
    corsaro_log(__func__, corsaro, "could not open %s output file", 
		        plugin->name);
    goto err;
  }
~~~

Because the \ref plugins_dos plugin uses a generic output file, we simply used
the \ref corsaro_io_prepare_file function as described in \ref arch_io. We store
the returned \ref corsaro_file pointer in the `state` structure for later use.
If the file could not be opened, we issue a log message and jump to the `err`
block to clean up and return an error code.

These are the common steps that nearly every plugin will follow when
initializing for output. At this time you should also establish any other state
required. For example, the \ref plugins_dos plugin initializes a hash table and
stores a pointer in the state structure:

~~~
state->attack_hash = kh_init(av);
~~~

close_output {#close_output}
------------

The _close_output_ function should free any state that was established in the
_init_output_ function (or during processing). The _close_output_ function
should also be able to free partial state - i.e. when an error occurs during
initialization, and as such some (or all) of the state is unallocated. The
corresponding _close_output_ function to the _init_output_function shown above
for \ref plugins_dos plugin is as follows:

~~~{.c}
int corsaro_dos_close_output(corsaro_t *corsaro)
{
  struct corsaro_dos_state_t *state = STATE(corsaro);

  if(state != NULL)
    {
      if(state->attack_hash != NULL)
	{
	  kh_free(av, state->attack_hash, &attack_vector_free);
	  kh_destroy(av, state->attack_hash);
	  state->attack_hash = NULL;
	}

      if(state->outfile != NULL)
	{
	  corsaro_file_close(corsaro, state->outfile);
	  state->outfile = NULL;
	}
      corsaro_plugin_free_state(corsaro->plugin_manager, PLUGIN(corsaro));
    }
  return 0;
}
~~~

It starts by using the `STATE` macro to retrieve a pointer to the state
structure stored for this plugin, if this pointer is not `NULL`, it then
proceeds to free the hashtable and output file (if they are still
allocated). The output file is closed using the \ref corsaro_file_close function
described in the \ref arch_io section. Once the state has been free, the state
structure itself is freed by calling \ref corsaro_plugin_free_state and passing
a pointer to the plugin manager, and to 'itself' - the structure for the current
plugin.

start_interval {#start_interval}
--------------

The _start_interval_ function is used by Corsaro to inform a plugin that a new
interval (see the \ref arch_intervals section) has begun. The plugin may need to
initialize some per-interval state at this point. It is perfectly acceptable to
ignore a _start_interval_ event (i.e. simply return `0`) if the plugin has no
use for it.

end_interval {#end_interval}
------------

The _end_interval_ function is used by Corsaro to inform a plugin that the
current interval is ending. The plugin may need to write out aggregated data
(using the Corsaro \ref arch_io) for the interval, and possibly free some
per-interval state at this point. As with the _start_interval_ event, a plugin
may safely ignore an _end_interval_ event.

process_packet {#process_packet}
--------------

The _process_packet_ event is likely where the majority of a plugin's analysis
code will go. The function is called once for every packet that Corsaro
processes. Corsaro provides two arguments to the function - a pointer to the
current \ref corsaro state structure, from which the plugin and plugin state
structures can be retrieved, and also a pointer to a \ref corsaro_packet
structure.

A \ref corsaro_packet is a lightweight wrapper around a _libtrace_ packet and a
\ref corsaro_packet_state structure. The _libtrace_ packet encapsulates the
actual packet that was captured, and all the regular _libtrace_ API functions
may be used on it. See the
[libtrace API documentation](http://research.wand.net.nz/software/libtrace-docs/html/libtrace_8h.html)
for more details. Corsaro provides a simple macro, \ref LT_PKT, which
dereferences the \ref corsaro_packet pointer to allow access to the _libtrace_
packet.  For example, the \ref plugins_dos plugin uses this simple check to
ensure that there are no non-IPv4 packets in the trace:

~~~{.c}
  libtrace_packet_t *ltpacket = LT_PKT(packet);
  uint16_t ethertype;
  uint32_t remaining;
  
  ...
  
  /* check for ipv4 */
  if((temp = trace_get_layer3(ltpacket, &ethertype, &remaining)) != NULL &&
     ethertype == TRACE_ETHERTYPE_IP)
    {
      ip_hdr = (libtrace_ip_t *)temp;
    }
  else
    {
      /* not an ip packet */
      corsaro_log(__func__, corsaro, "non-ip packet found (ethertype: %x)", 
		ethertype);
      return 0;
    }
~~~

The \ref corsaro_packet_state structure which is contained in the \ref
corsaro_packet structure is used to pass meta-data about a packet down the
plugin chain. Currently, \ref corsaro_packet_state structure (in \ref
corsaro_int.h) must be altered to store new meta-data. This will likely
change in a future version.  To make use of the packet state, a plugin would add
an appropriate field to the structure, and then after some analysis of a packet,
set the field to an appropriate value. The state is then passed down the plugin
chain, and successive plugins can leverage the meta-data derived by the earlier
plugin.

For example, if we had some code to perform geolocation of IP addresses, we could
augment each packet with the latitude and longitude of the source. First we
would add fields to the \ref corsaro_packet_state structure:

~~~{.c}
typedef struct corsaro_packet_state
{
  ...
  
  /* The latitude of the source IP address (using xxx geolocation) */
  int16_t latitude;
  /* The longitude of the source IP address (using xxx geolocation) */
  int16_t longitude;
  
} corsaro_packet_state_t;
~~~

Then in the _process_packet_ function, our geolocation plugin would take the
packet provided, find the geolocation data for the source address, and store the
latitude and longitude in the packet state structure:

~~~{.c}
int corsaro_dos_process_packet(corsaro_t *corsaro, 
			     corsaro_packet_t *packet)
{
  ...
  int16_t latitude;
  int16_t longitude;

  /* code to do geolocation here... */

  packet->state.latitude = latitude;
  packet->state.longitude = longitude;
  
  return 0;
}
~~~

Implementing the Corsaro-In API   {#tutorials_plugin_corsaro_in}
===============================

_Under Development_

This section describes each of the functions that must be implemented to fully
comply with the \a corsaro-in API. Each function has some code snippets that
are almost always used. The actual function implementations will have \a
corsaro_<plugin_name>_ prefixed to the names.

This section assumes that the \a corsaro-in API has already been fully
implemented (in order to have generated the data), but it is theoretically
possible to only implement the \ref alloc_output function from that API and
still have the \a corsaro-in API function correctly.

The required functions are:
1. \ref probe_filename
2. \ref probe_magic
3. \ref init_input
4. \ref close_input
5. \ref read_record
6. \ref read_global_data_record

probe_filename {#probe_filename}
--------------

@@ theres a nice function in corsaro_plugin.c that will probably do this for you
if you followed the conventions about how to name the output file

probe_magic {#probe_magic}
-----------

@@ provide the code snippet to demonstrate the peek function

init_input {#init_input}
----------

close_input {#close_input}
-----------

read_record {#read_record}
-----------

read_global_data_record {#read_global_data_record}
-----------------------
