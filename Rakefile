#
# Copyright:: Copyright (c) 2013 Opscode, Inc.
# License:: Apache License, Version 2.0
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

require 'erb'
require 'ostruct'
require 'pathname'
require 'rake/clean'
require 'rubygems/dependency'
require 'rubygems/dependency_installer'

CLEAN.include('../tmp/*')
CLEAN.include('dist/gems')
CLEAN.include('lib/gems')
CLEAN.include('../*-obj/ship/*.tgz')
CLEAN.include('release/chef.manifest')
CLEAN.include('release/netconf.manifest')
CLOBBER.include('../*-obj')

GEM_SOURCE_ROOT = "dist/gems"
GEM_BUILD_ROOT = "lib/gems"

# Make the Chef and Netconf versions configurable. By default the
# latest version will be installed
CHEF_VERSION = ENV['CHEF_VERSION'] || "latest"
NETCONF_VERSION = ENV['NETCONF_VERSION'] || "latest"

# Force Nogogiri to compile against system libxml2/libxslt
ENV['NOKOGIRI_USE_SYSTEM_LIBRARIES'] = "true"

desc "download and generate a manifest for a Rubygem"
task :manifest_gems => [:clean] do

  # package_id identifies the package uniquely. You can use any value
  # from 1 to 31 for package-id. 0 should be considered reserved. The
  # value should be unique among all your manifests. There should never
  # be more than one package from the same provider with the same
  # package ID. (This does not apply to packages with the same name, but
  # different versions. These must use the same package ID in order to
  # support future upgrades.)
  #
  # we'll start at 2 since the Ruby package's id is 1
  package_id_index = 2

  {
    "chef" => CHEF_VERSION,
    "netconf" => NETCONF_VERSION
  }.each_pair do |gem_name, gem_version|

    # create a sandboxed gem repository for this gem and it's deps
    gem_install_dir = "#{GEM_SOURCE_ROOT}/#{gem_name}"
    gem_build_dir = "#{GEM_BUILD_ROOT}/#{gem_name}"

    # program_id is a number ranging from 0 to 63 and increasing with each
    # binary added to the manifest. The number must be unique among all
    # programs with a given package ID
    program_id_index = 0

    manifest_lines = []
    # first line contains the package_id
    manifest_lines << "/set package_id=#{package_id_index}"

    ##############################################
    # Install Gem (and it's dependencies) using
    # the Rubygems ruby API
    ##############################################

    installed_specs = gem_installer(File.expand_path(gem_install_dir)).install(gem_dependency(gem_name, gem_version))
    specs_with_native_extension = installed_specs.find_all{|spec| spec.extensions.any? }
    specs_with_executables = installed_specs.find_all{|spec| spec.executables.any? }

    ##############################################
    # Process Standard Gem Files
    ##############################################

    file_list = FileList.new("#{gem_install_dir}/**/*") do |fl|
      # exclude directories
      fl.exclude{|f| File.directory?(f) }
      # exclude native extensions (we'll recompile them with the SDK)
      fl.exclude(/\.so/)
      # exclude executables (we'll process these separately)
      fl.exclude("**/**/bin/*")
    end

    file_list.each do |file|
      # given `dist/gems/chef/specifications/json-1.7.7.gemspec`
      # this should return `specifications/json-1.7.7.gemspec`
      path_minus_base = file.match(/#{gem_install_dir}\/(.*)/)[1]
      manifest_lines << "%TOPDIR%/#{file} store=%INSTALLDIR%/lib/ruby/gems/1.9.1/#{path_minus_base} mode=444"
    end

    ##############################################
    # Process Native Extensions
    ##############################################

    banner "Processing native extensions for #{gem_name}"
    manifest_lines << "######## NATIVE EXTENSIONS ########"

    specs_with_native_extension.each do |spec|

      puts "       > #{spec.name}"

      # dist/gems/chef/gems/json-1.7.7
      spec_install_path = "#{gem_install_dir}/gems/#{spec.name}-#{spec.version}"
      # lib/gems/chef/gems/json
      spec_build_root = "#{gem_build_dir}/#{spec.name}"

      spec.extensions.each do |e|

        # e = "ext/json/ext/generator/extconf.rb"

        # `ext/json/ext/generator`
        relative_source_path = e.match(/(.*)\/extconf.rb/)[1]
        # `gems/chef/gems/json-1.7.7/ext/json/ext/generator`
        extension_source_path = "#{spec_install_path}/#{e.match(/(.*)\/extconf.rb/)[1]}"
        # `json/ext/generator`
        relative_install_path = relative_source_path.match(/ext\/(.*)/)[1]
        # turns `yajl` into `yajl/yajl`
        relative_install_path = "#{relative_install_path}/#{relative_install_path}" unless relative_install_path =~ /\//
        # `generator`
        extension_name = relative_install_path.split(/\//).last

        # `lib/gems/chef/gems/json/generator`
        extension_build_root = "#{spec_build_root}/#{extension_name}"

        # Cleanup *.so generated by the `gem install`
        FileUtils.rm Dir.glob("#{extension_source_path}/*.so")

        makefile_content = render_erb(File.read("templates/extension_makefile.erb"), {
          :name => extension_name,
          :source_path => extension_source_path,
          :sources => FileList.new("#{extension_source_path}/*.c").map{|f| File.basename(f) }
        })

        # write out a Makefile for the extension
        ext_makefile_path = Pathname.new("#{extension_build_root}/Makefile")
        ext_makefile_path.dirname.mkpath
        ext_makefile_path.open("w") do |makefile|
          makefile.puts(makefile_content)
        end

        # add manifest line for the native extension
        manifest_lines << "%TOPDIR%/#{extension_build_root}/#{extension_name}.so store=%INSTALLDIR%/lib/ruby/gems/1.9.1/gems/#{spec.name}-#{spec.version}/lib/#{relative_install_path}.so mode=444"
      end

      # Write out a Makefile for the dep
      #
      # lib/gems/chef/json/generator/Makefile
      create_subdirectory_makefile("#{spec_build_root}/Makefile")
    end

    # Write out a Makefile for the gem
    #
    # lib/gems/chef/json/Makefile
    create_subdirectory_makefile("#{gem_build_dir}/Makefile")

    ##############################################
    # Process Gem Executables
    ##############################################

    banner "Processing executables for #{gem_name}"
    manifest_lines << "######## EXECUTABLES ########"
    specs_with_executables.each do |spec|
      spec.executables.each do |executable|
        puts "       > #{executable}"

        # Add a line for the gem-level bindir
        executable_path = "gems/#{spec.name}-#{spec.version}/#{spec.bindir}/#{executable}"
        manifest_line = "%TOPDIR%/#{gem_install_dir}/#{executable_path} store=%INSTALLDIR%/lib/ruby/gems/1.9.1/#{executable_path} mode=555 program_id=#{program_id_index += 1}"
        manifest_lines << manifest_line

        # Add a line for the top-level Rubygems bindir, we'll install
        # this one at the main sbin directory for the package
        executable_path = "bin/#{executable}"
        manifest_line = "%TOPDIR%/#{gem_install_dir}/#{executable_path} store=%INSTALLDIR%/bin/#{executable} mode=555 program_id=#{program_id_index += 1}"
        manifest_lines << manifest_line
      end

    end

    # write out the manifest file
    manifest_path = File.expand_path("release/#{gem_name}.manifest")
    banner "Writing out manifest file for #{gem_name} at #{manifest_path}"
    File.open(manifest_path, "w") do |manifest|
      manifest.puts(manifest_lines.join("\n"))
    end
  end

  # Write out a Makefile for all gems
  #
  # lib/gems/Makefile
  create_subdirectory_makefile("#{GEM_BUILD_ROOT}/Makefile")
end

desc "build all the things"
task :compile do
  system "mk etc" || raise
  system "mk-powerpc lib" || raise
  system "mk-powerpc sbin" || raise
end

desc "package all the things"
task :package => [:manifest_gems, :compile] do
  Dir.chdir "release"
  system "mk-powerpc chef-bundle"
end

private

def banner(msg = nil)
  puts "\n-----> #{msg}\n\n"
end

def gem_installer(install_path=nil, opts={:env_shebang => true})
  if install_path
    banner "Installing gems to #{install_path}"
    opts.merge!(:install_dir => install_path)
  end
  Gem::DependencyInstaller.new(opts)
end

def gem_dependency(name, version=nil)
  version = nil if version.to_s == "latest"
  Gem::Dependency.new(name, version)
end

def render_erb(template, locals)
  ERB.new(template).result(OpenStruct.new(locals).instance_eval { binding })
end

def create_subdirectory_makefile(path)
  gem_makefile_path = Pathname.new(path)
  makefile_content = render_erb(File.read("templates/subdir_makefile.erb"), {
    :subdirectories => gem_makefile_path.parent.children(false)
  })
  gem_makefile_path.open("w") do |makefile|
    makefile.puts(makefile_content)
  end
end
