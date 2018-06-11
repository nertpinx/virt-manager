# Design document for virtuned implementation

This document serves as a place to have open discussion about the design of so
called *libvirt profiles*, a.k.a. **virtuned**. The project should provide a
common ground to few problems which are currently being fixed on different
layers (mostly up the stack). Starting from the small things first, virtuned is
currently supposed to manage policies for creating XML definitions for VMs
(domains).

## Brief specification of functionality

Currently virtuned aims to provide a consistent way of applying profiles to
libvirt VM definitions.  That way management applications don't need to
duplicate the implementation in their codebases.

### Functions

As a starting point virtuned exposes one function.  As input the function
accepts a VM definition with the only restriction being that it is a libvirt
domain XML.  However it doesn't have to be complete.  The function applies all
relevant profiles to that XML and produces a complete libvirt domain XML.

The outcome of this is twofold:
- Every libvirt domain XML is already working virtuned XML.
- Applications can select, by arbitrarily small steps, how much functionality
  they want to use from virtuned.

The main advantage of this approach is that applications can adopt the new
functionality incrementally by small steps that they themselves can choose,
starting from nothing.  Simply plugging virtuned into the process of domain XML
creation shouldn't change anything until you explicitly start using profiles.

### API endpoints ###

For now the API will be exposed as:

1. Python module - trivial if we're basing it on virt-manager codebase which is
   using python
2. RESTful service - this will just be a separate file handing the glue between
   the python module and the different way of API exposure.  For that reason
   (and simplicity's sake) this could be outsourced (to
   e.g. [hug](https://www.hug.rest/)), at least for now<sup
   id='fn1'>[[1]](#fn1d)</sup>.

This will make it usable by most known projects and additional APIs can be added
later on without much friction (similarly to the REST API).  If virt-manager's
codebase is used as a base, then it will also simplify the exposure of other
parts of virt-manager under that RESTful service, for example virt-xml and
virt-install.  As far as I know this is something cockpit project would like a
lot <sup id='fn2'>[[2]](#fn2d)</sup>.

<sub id='fn1d'>1. <a href='#fn1'>**^**</a>
With hug we'll get CLI almost automatically as well.
</sub>  
<sub id='fn2d'>2. <a href='#fn2'>**^**</a>
This is not a strict requirement for virtuned, just a helpful side-effect.
</sub>

## Data specifications

There are several data formats that need to be specified.  What is discussed
below is mainly:
- **Format of the profiles** - Syntax of the format itself
- **Profile behaviours** - Possible changes profiles can do to VM definitions
- **Connection between profiles and VM definitions** - how to select changes
  which should be applied
- **Behaviour of the above connection** - How to specify multiple profiles and
  under what circumstances do they mutually exclude.

For a TL;DR example, see [Full Example](#full-example) below.

### Profile specification

The profile itself can influence the VM definition in three different ways:

- Requesting a specific part of the XML to (not) exist.  This includes:
  - Adding a specific XML snippet
  - Making sure specific XML snippet exists (without necessarily adding it)
  - Removing a specific XML snippet
- Setting default for existing parts of the XML (setting if unset)

Due to the fact that the profiles will influence how the resulting XML will look
like, virtuned profiles use XML as well, however that does not prevent the
support for other formats to be added later on.

Simple profile can look like this:

``` xml
<profile name='add-qxl'>
  <add>
    <devices>
      <video>
        <model type='qxl'/>
      </video>
    <devices>
  </add>
</profile>
```

The above example will request a video card with model QXL to exist in the VM
definition.  The precise outcome of this depends on the existing devices in the
VM definition:

- **VM has no video device:** the XML snippet (`qxl` video card) will simply be
  added to the list of devices.
- **VM has video device with no model specified:** Just fill in the video model
  for the existing video card.
- **VM has video device with different model:** Add one more video device with
  the specified model since multiple video cards are perfectly fine.

The above is very concrete example, but it can be very easily and efficiently
generalized for any `<add/>` sub-element.  The only information which is
required for said generalization is the knowledge of libvirt's domain XML
format.  This could be one of the reasons for virtuned to be spun off of
virt-manager's codebase (since most of that information is already there).  The
other option would be using
[libvirt-go-xml](https://libvirt.org/git/?p=libvirt-go-xml.git) as that should
have enough information for this as well <sup id='fn3'>[[3]](#fn3d)</sup>.

As mentioned above this is not the only type of action that the profile format
supports.  Here is the proposed list of actions with optional attributes:

- **`add`** - Make sure such XML snippet exists.  Can have attribute `multiple`
  with the following values:
  - **`yes`** - Unconditionally add the snippet if it can exist multiple times
    (the lowest level that can exist multiple times to be precise) or fail if it
    cannot (machine type)
  - **`no`** - Adjust existing part of the XML so that it matches the
    requirements from the snippet, overriding values if needed.
  - **`auto`** (default) - Try adjusting existing part of the XML so that it
    matches the requirements, but only override values if there is no part of
    that snippet that could be specified multiple times.  If any part of it can
    be specified multiple times, then find the lowest such part and append that.
- **`remove`** - Make sure such XML snippet does not exists.  All matching XML
  snippets (even if they have more attributes or sub-elements) will be removed.
- **`default`** - If VM definition has an XML snippet which does fit this
  description except some values not existing, then fill in those values.  This
  can be used for example for default device model types or machine types.

The `<add multiple='X'/>` is just a naming and it can be changed in any way that
suits others, for example instead of having:

``` xml
<add multiple='auto'/>
<add multiple='yes'/>
<add multiple='no'/>
```

There could be:

``` xml
<append/>
<set/>
<force/> <!-- or <replace/> -->
```

All action elements can have optional attribute `constraint` with the following
possible values:
- **`soft`** (default) - Profiles with higher priority can override this value.
  This is the default and should be used whenever it is not absolutely necessary
  for the XML snippet to be kept.
- **`hard`** - If profile with higher priority needs to override this value,
  then error out.  This should be selected only when it is absolutely necessary
  for the XML snippet to exist in this way.  For example in the following cases:
  - The system would be unstable.
  - Data corruption might occur.
  - Other parts of the profile would cause harm without this set.

Yet another simple profile can look like this:
``` xml
<profile name='some-interesting-things'>
  <add>
    <iothreads>2</iothreads>
  </add>
  <add>
    <devices>
      <disk device='cdrom'>
    </devices>
  </add>
  <add multiple='yes'>
    <devices>
      <redirdev bus='usb' type='spicevmc'/>
      <redirdev bus='usb' type='spicevmc'/>
    </devices>
  </add>
  <remove type='hard'>
    <features>
      <apic/>
    </features>
  </remove>
  <defaults>
    <devices>
      <interface>
        <model type='virtio'/>
      </interface>
    <devices>
  </defaults>
</profile>
```

This profile consists of various actions and has the following implications for
the VM definition:

- The VM should have 2 I/O Threads (profiles with higher priority can override
  this setting)
- The VM should have a CDROM drive.  It will not be added multiple times if it
  already exists.
- Two spice redirdev ports will be added to the VM definition.  If there were
  some existing ones, these will be added<sup id='fn4'>[[4]](#fn4d)</sup>.
- The VM must not have APIC (cannot be overridden)
- Any interface should default to virtio model type.  That means model will be
  set to `virtio` unless already specified.

There are some open questions related to more actions being specified, however
they should be limited to minimum.

<sub id='fn3d'>3. <a href='#fn3'>**^**</a>
Actually maybe even more since virt-manager's info is also incomplete.
</sub>  
<sub id='fn4d'>4. <a href='#fn4'>**^**</a>
Without the `multiple='yes'` this would mean that **at least** 2 such ports
should exist.
</sub>

### VM <-> Profile connection

Not all profiles need to be applied to all VM definitions.  In order to select
only the relevant ones we need to specify the connection between the VM
definition and the profile.  That can can be done in multiple different ways
depending on the preference, however each approach has pros and cons so they are
discussed in this section.

Since multiple profiles can be applied to the same VM definition at the same
time, there also needs to be a way to deal with conflicts.  Even though this
issue seems orthogonal to the connection itself, it can be dealt with in
different ways depending on the connection specification used.  What is proposed
below are two ways how to handle the connection with a way how to deal with
profile clashes together with two ways that were removed from the consideration
(just to makes sure the decisions are covered for future observers).

#### Selectors in profiles

Similarly to KubeVirt's approach to [VM
Presets](https://github.com/kubevirt/kubevirt/blob/master/docs/vm-presets.md)
this is something that has a great power.  Each profile specification includes a
selector based on which that particular profile will (not) be selected.

Multiple profiles clash and error out in case they cannot be combined.  For this
we propose a solution in the later section.

Example:

``` xml
<profile name='add-qxl-for-spice'>
  <match>
    <devices>
      <graphics type='spice'/>
    <devices>
  </match>
  <add>
    <devices>
      <video>
        <model type='qxl'/>
      </video>
    <devices>
  </add>
</profile>
```

This profile is similar to the one in [Profile
specification](#profile-specification) with one difference, which is a
`<match/>` element.   That element includes a condition under which the profile
actions will be executed.  In this particular case the profiles says that a QXL
video card should be present in case the VM has a SPICE graphics device.

These matches might include any part of the XML, even metadata, so this can
match guest OS (if provided as part of the VM metadata).

For example this condition:

``` xml
<match>
  <metadata>
    <myapp:myapp>
      <myapp:guest os_type='windows'/>
    </myapp:myapp>
  <metadata>
</match>
```

would be matched on this VM definition:

``` xml
<domain>
  <name>Win10</name>
  <metadata>
    <myapp:myapp xmlns:myapp='http://example.org/myapp'>
      <myapp:guest os_type='windows' os_version='10'/>
    </myapp:myapp>
  </metadata>
  ...
</domain>
```

As you can see the metadata used for the condition don't need to be virtuned's
specific metadata, but rather any management applications metadata.

#### Profile names in VM definitions

Yet another way how virtuned selects profiles to apply is a list of profiles
which is specified as part of the VM definition itself.  For example:

```xml
<domain>
  <name>asdf</name>
  <metadata>
    <virtuned:virtuned xmlns:virtuned='http://virt-manager.org/schemas/virtuned/0.1'>
      <virtuned:profile name='throughput' priority='50'/>
      <virtuned:guest os_type='centos' os_version='7' priority='99'/>
      <virtuned:profile name='stuff-that-makes-sense' priority='0'/>
      <virtuned:profile name='start-small'/>
    </virtuned:virtuned>
  </metadata>
  <memory unit='GiB'>4</memory>
  <devices>
    <disk type='file' device='disk'>
      <source path='/path/to/some/disk.img'/>
    </disk>
  </devices>
  <graphics type='spice'/>
</domain>
```

This example shows how each VM definition can select the profiles to be applied.
The advantage with this is that each profile can have a different priority based
on the VM's purpose.  That way you can select whether throughput-performance has
higher priority than the fact that it should be working with centos7 or the
other way around.  Multiple profiles clash and error out in case they cannot be
combined and have the same priority (or the one with lower priority has a hard
constraint using `type='hard'`).  Priority is optional and defaults to `0`.  The
part of the VM definition which is already specified (in this case it is memory
size and a disk device) is treated as having priority `99`.

The code complexity in this case is pretty high, however it balances out by the
usability and versatility of the profiles.

Moreover, this approach is **not** mutually exclusive with the previous one.
That is unless the previous one adds support for priorities in profile metadata.
That, however, would not be as versatile.

#### Approaches that were not selected

The following approaches were considered, but not selected.  They are mentioned
here so that the decision making process is documented.

##### ~~Profile inheritance~~

This approach, which is also taken by [tuned](http://tuned-project.org/) deals
very cleanly with multiple profiles.  That is thanks to the inheritance that
tuned profiles can utilize.  It does not implement a multiple inheritance for
obvious reasons; there will be no conflicts between profiles.

This has one major drawback.  For each orthogonal setting there is new dimension
in the matrix.  Since most of the possibilities need to be accounted for the
number of them explodes with any added dimension (type of setting).  This is
fine for tuned since the number of dimensions is not that large.  For virtuned,
however, the profile names would either get gigantic and complicated.  Imagine
trying to understand something like `sane-windows-pinned-small-some_defaults`
or, even better, coming up with the profile name while specifying the parts in
the right order.

##### ~~API function arguments~~ #####

Yet another option would be to specify the profiles (and their metadata, such as
priority) as an argument to the API call.  This, however, adds burden on the API
level without much of a usability.  This is something that's not very desired.

## Summary

### Full Example

For those who would like to see full example here is a completely artificial
one.  Hopefully this will help to complete the picture.

So if you have the following VM definition:

``` xml
<domain>
  <name>asdf</name>
  <metadata>
    <virtuned:virtuned xmlns:virtuned='http://virt-manager.org/schemas/virtuned/0.1'>
      <virtuned:guest os='centos7'/>
      <virtuned:profile name='spice-stuff'/>
      <virtuned:profile name='sensible-defaults'/>
      <virtuned:profile name='hyperv-defaults'/>
      <virtuned:profile name='myapp-defaults' priority='50'/>
    </virtuned:virtuned>
  </metadata>
  <memory unit='GiB'>4</memory>
  <devices>
    <disk type='file' device='disk'>
      <source path='/path/to/some/disk.img'/>
    </disk>
    <graphics type='spice'/>
    <interface/>
  </devices>
</domain>
```


And these profiles:

``` xml
<profile name='spice-stuff'>
  <match>
    <devices>
      <graphics type='spice'/>
    <devices>
  </match>
  <add>
    <devices>
      <video>
        <model type='qxl'/>
      </video>
    <devices>
  </add>
  <add multiple='yes'>
    <devices>
      <redirdev bus='usb' type='spicevmc'/>
      <redirdev bus='usb' type='spicevmc'/>
    </devices>
  </add>
</profile>
```

``` xml
<profile name='sensible-defaults'>
  <add multiple='no'>
    <devices>
      <video>
        <model type='qxl'/>
      </video>
    <devices>
  </add>
  <defaults>
    <devices>
      <interface>
        <model type='virtio'/>
      </interface>
    <devices>
  </defaults>
</profile>
```

``` xml
<profile name='hyperv-defaults'>
  <match>
    <metadata>
      <virtuned:virtuned>
        <virtuned:guest os_type='windows'/>
      </virtuned:virtuned>
    <metadata>
  </match>
  <add>
    <features>
      <hyperv>
        <relaxed state="on"/>
        <vapic state="on"/>
        <spinlocks state="on" retries="8191"/>
      </hyperv>
    </features>
  </add>
</profile>
```

```xml
<profile name='myapp-defaults'>
  <defaults type='hard'>
    <devices>
      <graphics type='spice'/>
      <interface>
        <model type='e1000e'/>
      </interface>
    <devices>
  </defaults>
</profile>
```

The output could look similarly to this:

``` xml 
<domain>
  <name>asdf</name>
  <metadata>
    <virtuned:virtuned xmlns:virtuned='http://virt-manager.org/schemas/virtuned/0.1'>
      <virtuned:guest os='centos7'/>
      <virtuned:profile name='spice-stuff'/>
      <virtuned:profile name='sensible-defaults'/>
      <virtuned:profile name='hyperv-defaults'/>
      <virtuned:profile name='myapp-defaults' priority='50'/>
    </virtuned:virtuned>
  </metadata>
  <memory unit='GiB'>4</memory>
  <devices>
    <disk type='file' device='disk'>
      <source path='/path/to/some/disk.img'/>
    </disk>
    <interface>
      <model type='e1000e'/>
    </interface>
    <video>
      <model type='qxl'/>
    </video>
    <redirdev bus='usb' type='spicevmc'/>
    <redirdev bus='usb' type='spicevmc'/>
  </devices>
</domain>
```

### Open Questions

There are still couple of things to be discussed, which I will only cover
slightly <sup id='fn5'>[[5]](#fn5d)</sup>.

1. **How are profiles added**  
   They can be files in a filesystem, there could be separate API for defining
   profiles.  Some of them will most probably be shared in the repository
   together with virtuned, but applications need to be able to define their own
   ones.  Filesystem-based storage seems fine for usage in a container for
   example, but might not be usable for some deployments.

2. **The need for "variables"**  
   Probably not as input, but rather as an already existing part of the
   definition.  For example setting the number of interface queues to a number
   derived from the number of vCPUs.  This has a big potential and based on the
   way we're going to go there are also various ways how to approach that.

3. **Topology mirroring (automatic pinning)**  
   One of the future feature requests is for the possibility of mirroring the
   host CPU and memory topology of the host (or part of it) to the guest.  Even
   though this doesn't fit into a *profile*, it's still something that might
   just as well fit into virtuned.  We'll only need a different API and a way to
   gather information about the host.  Either using livbirt or getting that data
   as an input to the API function

4. **Idempotent profiles**  
   There could be a benefit if profiles were idempotent <sup
   id='fn7'>[[7]](#fn7d)</sup>.  From the design above the only action that does
   not make them idempotent is the `multiple add` action.  If the profiles are
   idempotent then the round-trip through virtuned maybe done when modifying
   existing VM definitions (adding a device).  If not, then we're probably going
   to need to mark applied profiles in the metadata so that users and management
   applications can see whether they were applied or not.
   
5. **Versioned profiles**  
   Profiles should be versioned (or at least those that are going to be part of
   the same repository as virtuned, but rather all of them) so that there is no
   confusion and we don't get bitten later on if we need to upgrade.

6. **Make profiles work with other actions**  
   Profiles should not be something that is only applied when creating the
   domain XML.  Profiles should be used when, for example, updating an XML, when
   hot/cold-(un)plugging a device etc.  However for this we first need the above
   bits to be working.  However this needs to be taken into account when
   discussing any features.

7. **Advanced match conditions**  
   Similarly to point number 2, match conditions might need more intelligence
   than just pre-existing pieces of definition.  It starts with, for example,
   the opposite (i.e. part of XML *not* existing), but more conditions might be
   needed.  For example if the domain has number of vCPUs greater than the
   number of I/O Threads plus 2.

8. **The intelligence of profile specifications**  
   One of the bigger questions is whether the processing virtuned is going to do
   should be generic or aimed at VM definitions.  Hopefully the latter is the
   right way to go.  That way the built-in intelligence can simplify the
   language used to specify profiles.  But it is worth mentioning that simple,
   (or "stupid" if you will) XML handling, on the other hand (such that treats
   the VM definition purely as any other XML) has its advantages as well.  For
   example the fact that the profiles are not limited in what they can do,
   although that means there needs to be more logic in them.

To better illustrate some of the ideas mentioned in the last point here are two
programs (or rather simple program and a simple script).

The first one is a program that takes information about the definition from the
XML specification itself (imagine that is taken from libvirt-go-xml as mentioned
<a href='#fn3'>before</a>).  It appends values for those that can be specified
multiple times and rewrites those that can only be specified once.  You can try
this directly in a [playground](https://play.golang.org/p/dC4ihgrnIww).

The second example is a script written for an executable named `xq`.  That is
part of Project [yq](https://yq.readthedocs.io/en/latest/) (which is a YAML
extension of the [jq](https://stedolan.github.io/jq/manual/) project) and it has
support for working with XML using very powerful `jq` scripts.  The syntax could
be made better or easier for our specific purpose, but other than that the
script (which represents a profile) should be pretty self-exaplanatory.  It also
support variables, math expressions, own functions, loadable modules and more.

So here is the first (definition-aware) program:

``` golang
package main

import (
	"encoding/xml"
	"fmt"
	"reflect"
	"strings"
)

type Test struct {
	XMLName xml.Name `xml:"domain"`
	S       string   `xml:"single,omitempty"`
	M       []string `xml:"multiple"`
}

func f(v interface{}, s string) {
	t := reflect.TypeOf(v).Elem()
	va := reflect.ValueOf(v).Elem()

	for i := 0; i < t.NumField(); i++ {
		f := t.Field(i)
		val := va.Field(i)

		if f.Offset == 0 {
			continue
		}

		name := strings.Split(f.Tag.Get("xml"), ",")[0]
		if !strings.HasPrefix(name, s) {
			continue
		}

		if f.Type.Kind() == reflect.Slice &&
			f.Type.Elem().Kind() == reflect.String {
			val.Set(reflect.Append(val, reflect.ValueOf(s)))
		} else if f.Type.Kind() == reflect.String {
			val.Set(reflect.ValueOf(s))
		}
	}
}

func main() {
	fields := [][]string{
		[]string{"sin", "mul"},
		[]string{"multip"},
		[]string{"s", "sing"},
		[]string{"multip", "multi", "sin", "mul", "single"},
	}

	for i := 0; i < len(fields); i++ {
		test := Test{}

		for j := 0; j < len(fields[i]); j++ {
			f(&test, fields[i][j])
		}

		buf, err := xml.MarshalIndent(test, "", "  ")
		if err != nil {
			fmt.Println("Error:", err)
		} else {
			if i != 0 {
				fmt.Println()
			}
			fmt.Println("Test", fields[i])
			fmt.Println(string(buf))
		}
	}
}
```

And here is the filter for the `xq` program (can be saved in a file and then
executed using `xq -x -f <file> input.xml`) which treats the definition as a
pure XML:

``` auto
if .domain."@type" | not then
    .domain."@type" = "kvm"
else . end |

if .domain.memory | not then
    .domain.memory = { "#text": "1024", "@unit": "KiB" }
else . end |

.domain.os.type."#text" = "hvm" |

if .domain.devices.controller | not then
    .domain.devices.controller += [{ "@type": "usb", "@index": 0, "@model": "none" }]
else . end |

if .domain.devices.memballoon | not then
    .domain.devices.memballoon = { "@model": "none" }
else . end
```

<sub id='fn5d'>5. <a href='#fn5'>**^**</a>
It feels like it's fobvrever-growing <sup>[[6]](#fn6d)</sup>, it already delayed
this PR for almost a week and multiplied the amount of the content few times.
</sub>  
<sub id='fn6d'>6. <a href='#fn6'>**^**</a>
Just like my to-do list.
</sub>  
<sub id='fn7d'>7. <a href='#fn7'>**^**</a>
Having the same effect even if applied multiple times.
</sub>
