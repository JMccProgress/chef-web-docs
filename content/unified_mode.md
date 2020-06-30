+++
title = "Unified Mode"
draft = false

aliases = ["/unified_mode.html"]

[menu]
  [menu.infra]
    title = "Unified Mode Guide"
    identifier = "chef_infra/cookbook_reference/resources/unified_mode.md Unified Modee"
    parent = "chef_infra/cookbook_reference/resources"
    weight = 50
+++

[\[edit on GitHub\]](https://github.com/chef/chef-web-docs/blob/master/content/unified_mode.md)

## Unified Mode

Unified mode is a setting which eliminates the distinction between the compile and converge modes
in the way that Chef recipe and resources are parsed.  The setting replaces Chef's two pass
parsing with single pass parsing, so that resources are executed as soon as they are declared.

This results in considerably clearer code, with considerably less Ruby knowledge required to understand
the order of operations.

### Unified Mode for Custom Resources

Unified mode can be turned on for a resource, with the `unified_mode` method, which takes a Boolean argument:

```ruby
unifed_mode true

provides :my_resource_name

action :run do
  # implementation code goes here
end
```

### Unified Mode for Recipe Code

This has not yet been written, so there is no setting for this yet.

### Unified Mode behavior for Sub-Resources

If a `unified_mode` resource calls a non-`unified_mode` resource the sub-resource is not executed in
`unified_mode`.  Sub-resources are not affected by the calling resource being in `unified_mode` or
not.

## Converting Resources to Unified Mode

### Identification of Sub-Resources

Sub-Resources are defined as resources which are declared in the Custom Resource's `resource_collection` and are
run in the converge phase.  Resources declared normally via the Chef language (e.g. `file`, `template`, etc) are
considered sub-resources.  Resources which are declared using `declare_resource` (e.g. `declare_resource(:file,...)`) are
also considered to be sub-resources.

Examples of sub-resource declarations:

``` ruby
action :run do
  file "/tmp/file1.xyz" do
    contents "mycontents"
  end

  declare_resource(:file, "/tmp/file2.xyz") do
    contents "mycontents"
  end
end
```

Resources forced to the compile phase (e.g. using `run_action(:create)`), resources built using `build_resource` (e.g.
`build_resource(:file, ...)` or resources built using the normal Ruby constructor (e.g. `Chef::Resource::File.new(..)`)
are not considered to be sub-resources since they are not doing work in the converge phase.

Examples of the use of Chef resources which do not count as sub-resources:

``` ruby
action :run do
  file "/tmp/file3.xyz" do
    contents "mycontents"
    action :nothing
  end.run_action(:create)

  build_resource(:file, "/tmp/file4.xyz") do
    contents "mycontents"
  end.run_action(:create)

  file_resource = Chef::Resource::File.new("/tmp/file5.xyz")
  file.resource.run_action(:create)
end
```

Any work done by ruby code inside a `converge_by` block is also clearly not a sub-resource, although sub-resources may
be declared inside of a `converge_by` block.  This is almost certainly a bug, and the kind of issue that `unified_mode`
is expected to eliminate, since the sub-resource is only declared when the `converge_by` block.

``` ruby
action_class do
  def should_create_file?
    ...
  end
end

action :run do
  if should_create_file?
    converge_by "creating a file" do
      # this counts as a subresource, and its action is deferred until later
      file "/tmp/file3.xyz" do
        contents "mycontents"
      end
    end
  end

  Chef::Log.warn "the file will be created after this line is output"
end
```

This pattern is usually a bug and the author intends the file resource to run when the `converge_by` block is being
parsed, which is not how Chef has behaved.  Unified Mode will fix this so that the execution order

### Resources Without Sub-Resources

Resources without any sub-resources do not strictly require any modification.  This is because they already do no work at
converge time, and the `resource_collection` of the Custom Resource is not being used.  These are resources written
"Chef-10 Style".

It is highly encouraged, however, that any resources which are being invoked at compile time through any of the methods
above (e.g. `Chef::Resource::File.new`) be converted to use the Chef language and become proper sub-resources.  This will
allow the Custom Resource to do notifications correctly.  The only exception being where the built resource is deliberately
not being placed in the resource collection to avoid causing the Custom Resource to fire notifications.

### Resource With Sub-Resources

Custom Resources that use Sub-Resources that are written with good style and do not mix proper Sub-Resources with out of order
ruby code and compile-time resources (most of them), do not require any modification.  This may, however, be difficult to determine
and proper testing will be essential.  Since all of the execution of the Sub-Resources move from the end of execution during the
compile time phase, into where they are actually declared resources may not behave exactly the same.

Once the resource has been converted to Unified Mode, additional cleanup can make the code considerably easier to read:

- Eliminate all `ruby_block` (or `whyrun_safe_ruby_block`) resources and just execute the code inside the block
- Eliminate all use of `lazy {}`
- In Chef Infra Client 16 calls to Chef::Log may be converted to log resources

### Notifications and Accumulators

The Accumulator pattern works unchanged.  Notifications to the `:root` run context still behave identically.  Since the compile and
converge phases of Custom Resources both fire in the converge time (typically) of the enclosing `run_context` the effect of eliminating
the separate compile and converge phases of the Custom Resource has no visible effect from the outer context.

