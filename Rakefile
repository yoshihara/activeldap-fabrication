# -*- coding: utf-8; mode: ruby -*-
#
# Copyright (C) 2011  Kouhei Sutou <kou@clear-code.com>
#
# This library is free software; you can redistribute it and/or
# modify it under the terms of the GNU Lesser General Public
# License as published by the Free Software Foundation; either
# version 2.1 of the License, or (at your option) any later version.
#
# This library is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the GNU
# Lesser General Public License for more details.
#
# You should have received a copy of the GNU Lesser General Public
# License along with this library; if not, write to the Free Software
# Foundation, Inc., 59 Temple Place, Suite 330, Boston, MA  02111-1307  USA

require 'English'

require 'jeweler'
require 'rake/testtask'
require "rake/clean"
require "yard"

$KCODE = "u" if RUBY_VERSION < "1.9"

base_dir = File.expand_path(File.dirname(__FILE__))
$LOAD_PATH.unshift(File.join(base_dir, 'lib'))
require 'active_ldap_fabrication'

project_name = "ActiveLdap Fabrication"

ENV["VERSION"] ||= ActiveLdapFabrication::VERSION::STRING
version = ENV["VERSION"]
spec = nil
Jeweler::Tasks.new do |_spec|
  spec = _spec
  spec.name = 'activeldap-fabrication'
  spec.version = version.dup
  spec.rubyforge_project = 'ruby-activeldap'
  spec.authors = ["Kouhei Sutou"]
  spec.email = ["kou@clear-code.com"]
  spec.summary = "ActiveLdap Fabrication is an ActiveLdap adapter for Fabrication"
  spec.homepage = "http://ruby-activeldap.rubyforge.org/"
  spec.files = FileList["lib/**/*",
                        "doc/text/**/*",
                        "COPYING",
                        "Gemfile",
                        "README.textile"]
  spec.test_files = FileList['test/test[_-]*.rb']
  spec.description = <<-EOF
    ActiveLdap Fabrication is an ActiveLdap adapter for
    Fabrication. It means that you can use Fabrication as
    fixture replacement for ActiveLdap.
  EOF
  spec.license = "LGPLv2 or later"
end

Rake::Task["release"].prerequisites.clear
Jeweler::RubygemsDotOrgTasks.new do
end

reference_base_dir = Pathname.new("doc/reference")
doc_en_dir = reference_base_dir + "en"
html_base_dir = Pathname.new("doc/html")
html_reference_dir = html_base_dir + spec.name
YARD::Rake::YardocTask.new do |task|
  task.options += ["--title", project_name]
  task.options += ["--charset", "UTF-8"]
  task.options += ["--readme", "README.textile"]
  task.options += ["--files", "doc/text/**/*"]
  task.options += ["--output-dir", doc_en_dir.to_s]
  task.options += ["--charset", "utf-8"]
  task.files += FileList["lib/**/*.rb"]
  task.files -= ["README.textile"]
end

task :yard do
  doc_en_dir.find do |path|
    next if path.extname != ".html"
    html = path.read
    html = html.gsub(/<div id="footer">.+<\/div>/m,
                     "<div id=\"footer\"></div>")
    path.open("w") do |html_file|
      html_file.print(html)
    end
  end
end

include ERB::Util

def apply_template(content, paths, templates, project_name, language)
  content = content.sub(/lang="en"/, "lang=\"#{language}\"")

  title = nil
  content = content.sub(/<title>(.+?)<\/title>/m) do
    title = $1
    templates[:head].result(binding)
  end

  content = content.sub(/<body(?:.*?)>/) do |body_start|
    "#{body_start}\n#{templates[:header].result(binding)}\n"
  end

  content = content.sub(/<\/body/) do |body_end|
    "\n#{templates[:footer].result(binding)}\n#{body_end}"
  end

  content
end

def erb_template(name)
  file = File.join("doc/templates", "#{name}.html.erb")
  template = File.read(file)
  erb = ERB.new(template, nil, "-")
  erb.filename = file
  erb
end

def rsync_to_rubyforge(spec, source, destination, options={})
  config = YAML.load(File.read(File.expand_path("~/.rubyforge/user-config.yml")))
  host = "#{config["username"]}@rubyforge.org"

  rsync_args = "-av --exclude '*.erb' --chmod=ug+w"
  rsync_args << " --delete" if options[:delete]
  remote_dir = "/var/www/gforge-projects/#{spec.rubyforge_project}/"
  sh("rsync #{rsync_args} #{source} #{host}:#{remote_dir}#{destination}")
end

def rake(*arguments)
  ruby($0, *arguments)
end

namespace :reference do
  translate_languages = [:ja]
  supported_languages = [:en, *translate_languages]
  html_files = FileList["#{doc_en_dir}/**/*.html"].to_a

  directory reference_base_dir.to_s
  CLOBBER.include(reference_base_dir.to_s)

  po_dir = "doc/po"
  pot_file = "#{po_dir}/#{spec.name}.pot"
  namespace :pot do
    directory po_dir
    file pot_file => [po_dir, *html_files] do |t|
      sh("xml2po", "--keep-entities", "--output", t.name, *html_files)
    end

    desc "Generates pot file."
    task :generate => pot_file
  end

  namespace :po do
    translate_languages.each do |language|
      namespace language do
        po_file = "#{po_dir}/#{language}.po"

        if File.exist?(po_file)
          file po_file => html_files do |t|
            sh("xml2po", "--keep-entities", "--update", t.name, *html_files)
          end
        else
          file po_file => pot_file do |t|
            sh("msginit",
               "--input", pot_file,
               "--output-file", t.name,
               "--locale", language.to_s)
          end
        end

        desc "Updates po file for #{language}."
        task :update => po_file
      end
    end

    desc "Updates po files."
    task :update do
      ruby($0, "clobber")
      ruby($0, "yard")
      translate_languages.each do |language|
        ruby($0, "reference:po:#{language}:update")
      end
    end
  end

  namespace :translate do
    translate_languages.each do |language|
      po_file = "#{po_dir}/#{language}.po"
      translate_doc_dir = "#{reference_base_dir}/#{language}"

      desc "Translates documents to #{language}."
      task language => [po_file, reference_base_dir, *html_files] do
        commands = []
        doc_en_dir.find do |path|
          base_path = path.relative_path_from(doc_en_dir)
          translated_path = "#{translate_doc_dir}/#{base_path}"
          if path.directory?
            mkdir_p(translated_path)
            next
          end
          case path.extname
          when ".html"
            command = "xml2po --keep-entities " +
              "--po-file #{po_file} --language #{language} " +
              "#{path} > #{translated_path} &"
            commands << command
          else
            cp(path.to_s, translated_path, :preserve => true)
          end
        end
        commands << "wait"
        sh(commands.join("\n"))
      end
    end
  end

  translate_task_names = translate_languages.collect do |language|
    "reference:translate:#{language}"
  end
  desc "Translates references."
  task :translate => translate_task_names

  desc "Generates references."
  task :generate => [:yard, :translate]

  namespace :publication do
    task :prepare do
      supported_languages.each do |language|
        raw_reference_dir = reference_base_dir + language.to_s
        prepared_reference_dir = html_reference_dir + language.to_s
        rm_rf(prepared_reference_dir.to_s)
        head = erb_template("head.#{language}")
        header = erb_template("header.#{language}")
        footer = erb_template("footer.#{language}")
        raw_reference_dir.find do |path|
          relative_path = path.relative_path_from(raw_reference_dir)
          prepared_path = prepared_reference_dir + relative_path
          if path.directory?
            mkdir_p(prepared_path.to_s)
          else
            case path.basename.to_s
            when /(?:file|method|class)_list\.html\z/
              cp(path.to_s, prepared_path.to_s)
            when /\.html\z/
              relative_dir_path = relative_path.dirname
              current_path = relative_dir_path + path.basename
              if current_path.basename.to_s == "index.html"
                current_path = current_path.dirname
              end
              top_path = html_base_dir.relative_path_from(prepared_path.dirname)
              paths = {
                :top => top_path,
                :current => current_path,
              }
              templates = {
                :head => head,
                :header => header,
                :footer => footer
              }
              content = apply_template(File.read(path.to_s),
                                       paths,
                                       templates,
                                       spec.name,
                                       language)
              File.open(prepared_path.to_s, "w") do |file|
                file.print(content)
              end
            else
              cp(path.to_s, prepared_path.to_s)
            end
          end
        end
      end
      File.open("#{html_reference_dir}/.htaccess", "w") do |file|
        file.puts("RedirectMatch permanent ^/#{spec.name}/$ " +
                  "#{spec.homepage}#{spec.name}/en/")
      end
    end
  end

  desc "Upload document to rubyforge."
  task :publish => [:generate, "reference:publication:prepare"] do
    rsync_to_rubyforge(spec, "#{html_reference_dir}/", spec.name)
  end
end

desc "Upload document and HTML to rubyforge."
task :publish => ["reference:publish"]

desc "Tag the current revision."
task :tag do
  sh("git", "tag", "-a", version, "-m", "release #{version}!!!")
end
