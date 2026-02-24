# Refer to https://github.com/plusjade/jekyll-bootstrap/blob/master/Rakefile

require "rubygems"
require 'rake'
require 'yaml'
require 'time'

SOURCE = "."
CONFIG = {
  'posts' => File.join(SOURCE, "_posts"),
  'drafts' => File.join(SOURCE, "_drafts"),
  'post_ext' => "md",
}

# Usage: rake post title="A Title" [date="2012-02-09"] [tags=[tag1,tag2]]
desc "Begin a new post in #{CONFIG['posts']}"
task :post do
  abort("rake aborted: '#{CONFIG['posts']}' directory not found.") unless FileTest.directory?(CONFIG['posts'])
  title = ENV["title"] || "new-post"
  tags = ENV["tags"] || "[]"
  slug = title.downcase.strip.gsub(' ', '-').gsub(/[^\w-]/, '')
  begin
    date = (ENV['date'] ? Time.parse(ENV['date']) : Time.now).strftime('%Y-%m-%d')
  rescue => e
    puts "Error - date format must be YYYY-MM-DD, please check you typed it correctly!"
    exit -1
  end
  filename = File.join(CONFIG['posts'], "#{date}-#{slug}.#{CONFIG['post_ext']}")
  if File.exist?(filename)
    abort("rake aborted!") if ask("#{filename} already exists. Do you want to overwrite?", ['y', 'n']) == 'n'
  end

  puts "Creating new post: #{filename}"
  open(filename, 'w') do |post|
    post.puts "---"
    post.puts "layout: post"
    post.puts "title: \"#{title.gsub(/-/,' ')}\""
    post.puts 'description: ""'
    post.puts "date: #{date}"
    post.puts "tags: #{tags}"
    post.puts "comments: true"
    post.puts "---"
  end
end # task :post

# Usage: rake draft title="A Title" [date="2012-02-09"] [tags=[tag1,tag2]]
desc "Begin a new post in #{CONFIG['drafts']}"
task :draft do
  abort("rake aborted: '#{CONFIG['drafts']}' directory not found.") unless FileTest.directory?(CONFIG['drafts'])
  title = ENV["title"] || "new-draft"
  tags = ENV["tags"] || "[]"
  slug = title.downcase.strip.gsub(' ', '-').gsub(/[^\w-]/, '')
  begin
    date = (ENV['date'] ? Time.parse(ENV['date']) : Time.now).strftime('%Y-%m-%d')
  rescue => e
    puts "Error - date format must be YYYY-MM-DD, please check you typed it correctly!"
    exit -1
  end
  filename = File.join(CONFIG['drafts'], "#{date}-#{slug}.#{CONFIG['post_ext']}")
  if File.exist?(filename)
    abort("rake aborted!") if ask("#{filename} already exists. Do you want to overwrite?", ['y', 'n']) == 'n'
  end

  puts "Creating new drafts: #{filename}"
  open(filename, 'w') do |draft|
    draft.puts "---"
    draft.puts "layout: post"
    draft.puts "title: \"#{title.gsub(/-/,' ')}\""
    draft.puts 'description: ""'
    draft.puts "date: #{date}"
    draft.puts "tags: #{tags}"
    draft.puts "comments: true"
    draft.puts "---"
  end
end # task :draft

# Usage: rake publish [file="draft-filename.md"]
desc "Promote a draft to a published post"
task :publish do
  abort("rake aborted: '#{CONFIG['drafts']}' directory not found.") unless FileTest.directory?(CONFIG['drafts'])

  drafts = Dir.glob(File.join(CONFIG['drafts'], "*.#{CONFIG['post_ext']}"))
  abort("No drafts found in #{CONFIG['drafts']}") if drafts.empty?

  if ENV["file"]
    chosen = File.join(CONFIG['drafts'], ENV["file"])
    abort("Draft not found: #{chosen}") unless File.exist?(chosen)
  else
    puts "Available drafts:"
    drafts.each_with_index { |f, i| puts "  #{i + 1}. #{File.basename(f)}" }
    print "Pick a draft (number): "
    choice = $stdin.gets.chomp.to_i
    abort("Invalid choice") if choice < 1 || choice > drafts.size
    chosen = drafts[choice - 1]
  end

  today = Time.now.strftime('%Y-%m-%d')
  basename = File.basename(chosen).sub(/^\d{4}-\d{2}-\d{2}-/, '')
  new_filename = File.join(CONFIG['posts'], "#{today}-#{basename}")

  content = File.read(chosen)
  content = content.sub(/^date:.*$/, "date: #{today}")

  File.write(new_filename, content)
  File.delete(chosen)
  puts "Published: #{new_filename}"
end # task :publish

# Usage: rake note title="Title" [tags=[tag1,tag2]]
desc "Create a short-form note post"
task :note do
  abort("rake aborted: '#{CONFIG['posts']}' directory not found.") unless FileTest.directory?(CONFIG['posts'])
  title = ENV["title"] || "new-note"
  tags = ENV["tags"] || "[]"
  slug = title.downcase.strip.gsub(' ', '-').gsub(/[^\w-]/, '')
  date = Time.now.strftime('%Y-%m-%d')
  filename = File.join(CONFIG['posts'], "#{date}-#{slug}.#{CONFIG['post_ext']}")

  if File.exist?(filename)
    abort("rake aborted!") if ask("#{filename} already exists. Do you want to overwrite?", ['y', 'n']) == 'n'
  end

  puts "Creating new note: #{filename}"
  open(filename, 'w') do |post|
    post.puts "---"
    post.puts "layout: post"
    post.puts "title: \"#{title.gsub(/-/, ' ')}\""
    post.puts "date: #{date}"
    post.puts "tags: #{tags}"
    post.puts "type: note"
    post.puts "comments: false"
    post.puts "---"
  end
end # task :note

desc "Launch preview environment"
task :preview do
  system "bundle exec jekyll serve -w --host 0.0.0.0"
end # task :preview
