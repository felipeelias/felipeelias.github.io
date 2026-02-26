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

desc "Convert images to WebP (requires cwebp and gif2webp from brew install webp)"
task :optimize_images do
  require 'fileutils'

  %w[assets assets/images].each do |dir|
    next unless File.directory?(dir)

    Dir.glob(File.join(dir, "*.{jpg,jpeg,png,gif}")).each do |src|
      dest = src.sub(/\.(jpg|jpeg|png|gif)$/i, '.webp')

      if File.exist?(dest)
        puts "  skip: #{dest} already exists"
        next
      end

      cmd = if src.end_with?('.gif')
              "gif2webp -lossy -q 80 #{src.shellescape} -o #{dest.shellescape}"
            else
              "cwebp -q 80 #{src.shellescape} -o #{dest.shellescape}"
            end

      if system(cmd)
        old_size = File.size(src)
        new_size = File.size(dest)
        savings = ((1 - new_size.to_f / old_size) * 100).round(1)
        puts "  #{src} -> #{dest} (#{savings}% smaller)"
      else
        puts "  FAILED: #{src}"
      end
    end
  end

  puts "\nDone. Remember to update references in posts and remove the originals."
end

# Usage: rake hashnode [file="2026-02-25-my-post.md"]
desc "Cross-post to Hashnode (posts with hashnode: true in front matter)"
task :hashnode do
  require 'net/http'
  require 'json'
  require 'uri'

  token = ENV["HASHNODE_TOKEN"]
  publication_id = ENV["HASHNODE_PUBLICATION_ID"]
  abort("HASHNODE_TOKEN is not set") unless token && !token.empty?
  abort("HASHNODE_PUBLICATION_ID is not set") unless publication_id && !publication_id.empty?

  site_url = YAML.safe_load_file("_config.yml", permitted_classes: [Time, Date])["url"]

  posts = Dir.glob(File.join(CONFIG['posts'], "*.#{CONFIG['post_ext']}")).select do |f|
    content = File.read(f)
    front_matter = YAML.safe_load(content.split("---", 3)[1], permitted_classes: [Time, Date])
    front_matter["hashnode"] == true
  end.sort

  abort("No posts with hashnode: true found in #{CONFIG['posts']}") if posts.empty?

  if ENV["file"]
    chosen = File.join(CONFIG['posts'], ENV["file"])
    abort("Post not found: #{chosen}") unless File.exist?(chosen)
  else
    puts "Eligible posts:"
    posts.each_with_index { |f, i| puts "  #{i + 1}. #{File.basename(f)}" }
    print "Pick a post (number): "
    choice = $stdin.gets.chomp.to_i
    abort("Invalid choice") if choice < 1 || choice > posts.size
    chosen = posts[choice - 1]
  end

  raw = File.read(chosen)
  parts = raw.split("---", 3)
  front_matter = YAML.safe_load(parts[1], permitted_classes: [Time, Date])
  markdown_body = parts[2].strip
  markdown_body = markdown_body.gsub(%r{(!\[[^\]]*\]\()(/[^)]+)\)}) { "#{$1}#{site_url}#{$2})" }

  basename = File.basename(chosen, ".#{CONFIG['post_ext']}")
  match = basename.match(/^(\d{4})-(\d{2})-(\d{2})-(.+)$/)
  abort("Cannot parse filename: #{basename}") unless match
  year, month, day, slug = match.captures
  canonical_url = "#{site_url}/#{year}/#{month}/#{day}/#{slug}.html"

  tags = Array(front_matter["tags"]).map do |t|
    { name: t, slug: t.downcase.gsub(/\s+/, "-") }
  end

  title = front_matter["title"]
  puts "Publishing \"#{title}\" to Hashnode..."
  puts "  Canonical URL: #{canonical_url}"

  uri = URI("https://gql.hashnode.com/")
  http = Net::HTTP.new(uri.host, uri.port)
  http.use_ssl = true

  create_draft_query = <<~GRAPHQL
    mutation CreateDraft($input: CreateDraftInput!) {
      createDraft(input: $input) {
        draft { id }
      }
    }
  GRAPHQL

  draft_input = {
    title: title,
    contentMarkdown: markdown_body,
    publicationId: publication_id,
    originalArticleURL: canonical_url,
    tags: tags,
    slug: slug
  }

  draft_body = { query: create_draft_query, variables: { input: draft_input } }.to_json
  draft_request = Net::HTTP::Post.new(uri.path, {
    "Content-Type" => "application/json",
    "Authorization" => token
  })
  draft_request.body = draft_body

  draft_response = http.request(draft_request)
  draft_result = JSON.parse(draft_response.body)

  if draft_result["errors"]
    abort("Failed to create draft: #{draft_result["errors"].map { |e| e["message"] }.join(", ")}")
  end

  draft_id = draft_result.dig("data", "createDraft", "draft", "id")
  abort("Failed to create draft: no draft ID returned") unless draft_id
  puts "  Draft created: #{draft_id}"

  publish_query = <<~GRAPHQL
    mutation PublishDraft($input: PublishDraftInput!) {
      publishDraft(input: $input) {
        post { id, url }
      }
    }
  GRAPHQL

  publish_body = { query: publish_query, variables: { input: { draftId: draft_id } } }.to_json
  publish_request = Net::HTTP::Post.new(uri.path, {
    "Content-Type" => "application/json",
    "Authorization" => token
  })
  publish_request.body = publish_body

  publish_response = http.request(publish_request)
  publish_result = JSON.parse(publish_response.body)

  if publish_result["errors"]
    abort("Failed to publish draft: #{publish_result["errors"].map { |e| e["message"] }.join(", ")}")
  end

  post_url = publish_result.dig("data", "publishDraft", "post", "url")
  puts "  Published: #{post_url}"
end

desc "Launch preview environment"
task :preview do
  system "bundle exec jekyll serve -w --host 0.0.0.0"
end # task :preview
