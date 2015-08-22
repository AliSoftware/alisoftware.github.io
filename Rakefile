drafts_dir = '_drafts'
posts_dir  = '_posts'

# rake post['my new post']
desc 'create a new post'
task :post, :title do |t, args|
  if args.title
    title = args.title
  else
    puts "Please try again. Remember to include the filename."
  end
  mkdir_p "#{posts_dir}"
  filename = "#{posts_dir}/#{Time.now.strftime('%Y-%m-%d')}-#{title.downcase.gsub(/[^\w]+/, '-')}.md"
  puts "Creating new post: #{filename}"
  File.open(filename, "w") do |f|
    f << <<-EOS.gsub(/^    /, '')
    ---
    layout: post
    title: #{title}
    date: #{Time.new.strftime('%Y-%m-%d %H:%M')}
    categories:
    ---

    EOS
  end

  # Automatically open the post in SublimeText
  system ("subl #{filename}")
end

# usage: rake draft['my new draft']
desc 'create a new draft post'
task :draft, :title do |t, args|
  if args.title
    title = args.title
  else
    puts "Please try again. Remember to include the filename."
  end
  mkdir_p "#{drafts_dir}"
  filename = "#{drafts_dir}/#{title.downcase.gsub(/[^\w]+/, '-')}.md"
  puts "Creating new draft: #{filename}"
  File.open(filename, "w") do |f|
    f << <<-EOS.gsub(/^    /, '')
    ---
    layout: post
    title: #{title}
    date: #{Time.new.strftime('%Y-%m-%d %H:%M')}
    categories:
    ---

    EOS
  end

# Uncomment the line below if you want the draft to automatically open in your default text editor
# system ("#{ENV['EDITOR']} #{filename}")
end

desc 'preview the site with drafts'
task :preview do
  puts "## Generating site"
  puts "## Stop with ^C ( <CTRL>+C )"
  system "jekyll serve --watch --drafts"
end


desc 'regenerate the site apple-touch/favicons images'
task :icons, :source do |_, args|
  unless args.source
    puts "Please try again. Remember to include the source file path."
    exit 1
  end
  sizes_list = {
    'apple-touch-icon' => [57, 60, 72, 76, 114, 120, 144, 152, 180],
    'favicon' => [16, 32, 96, 160, 192]
  }
  sizes_list.each do |prefix, sizes|
    sizes.each do |size|
      system "sips -z #{size} #{size} #{args.source} --out #{prefix}-#{size}x#{size}.png"
    end
  end
end
