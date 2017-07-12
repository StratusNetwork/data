require 'net/http'
require 'openssl'
require 'json'
require 'yaml'

RESET = "\e[0m"
RED   = "\e[31m"
YELO  = "\e[33m"
GREEN = "\e[32m"

task :default => [:validate]

desc "Show a list of all valid permissions"
task :perms do
    puts get(action_uri(:permissions)).body
end

desc "Validate local files with the server (but don't deploy anything)"
task :validate do
    uri = action_uri(:validate)
    puts "Validating models at #{uri}"
    response = post(uri, {files: read_model_files})
    puts YELO + response.body + RESET

    if response.code.to_i == 200
        puts "#{GREEN}Data looks valid#{RESET}"
    else
        puts "#{RED}Validation failed#{RESET}"
        raise "Validation failed"
    end
end

desc "Update local files to match what is currently deployed"
task :pull do
    uri = action_uri(:pull)
    puts "Pulling models from #{uri}"
    json = JSON.load(get(uri).body)
    puts YELO + json['log'] + RESET
    write_model_files(json['files'])
end

def read_model_files
    files = {}
    each_model_file do |path|
        files[path] = File.read(path)
    end
    files
end

def write_model_files(files)
    changed = false
    local = each_model_file.to_a

    in_model_dir do
        files.each do |path, content|
            if local.include? path
                old_yaml = YAML.load_file(path)
                new_yaml = YAML.load(content)
                if old_yaml != new_yaml
                    puts "Updating #{path}"
                    File.write(path, content)
                    changed = true
                end
            else
                puts "Creating #{path}"
                FileUtils.mkdir_p(File.dirname(path))
                File.write(path, content)
                changed = true
            end
        end

        (local - files.keys).each do |path|
            puts "Deleting #{path}"
            File.delete(path)
            changed = true
        end
    end

    puts "No changes" unless changed
end

def each_model_file
    if block_given?
        in_model_dir do
            Dir['**/*.yml'].each do |path|
                yield path
            end
        end
    else
        enum_for :each_model_file
    end
end

def in_model_dir
    Dir.chdir model_base_path do
        yield
    end
end

def model_base_path
    File.join(File.dirname(__FILE__), 'models')
end

def get(uri)
    make_http(uri).request(make_get(uri))
end

def post(uri, json)
    make_http(uri).request(make_post(uri, json))
end

def make_get(uri)
    request = Net::HTTP::Get.new(uri.path)
    add_headers(request)
    request
end

def make_post(uri, json)
    request = Net::HTTP::Post.new(uri.path)
    add_headers(request)
    request.body = json.to_json
    request
end

def add_headers(request)
    request.add_field('Content-Type', 'application/json')
    request.add_field('X-OCN-Key', api_key)
end

def make_http(uri)
    http = Net::HTTP.new(uri.host, uri.port)
    if uri.scheme == 'https'
        http.use_ssl = true
        http.verify_mode = OpenSSL::SSL::VERIFY_NONE
    end
    http
end

def action_uri(action)
    URI.parse("#{base_uri}/admin/data/#{action}")
end

def base_uri
    ENV['OCTC']
end

def api_key
    ENV['OCN_API_KEY'] or raise "Missing OCN_API_KEY environment variable"
end
