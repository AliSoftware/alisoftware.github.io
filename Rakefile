drafts_dir = '_drafts'
posts_dir  = '_posts'

# rake post['my new post']
desc 'create a new post'
task :post, :title do |_, args|
  if args.title
    title = args.title
  else
    abort "Please try again. Remember to include the filename."
  end
  mkdir_p "#{posts_dir}"
  filename = "#{posts_dir}/#{Time.now.strftime('%Y-%m-%d')}-#{title.downcase.gsub(/[^\w]+/, '-')}.md"
  puts "Creating new post: #{filename}"
  File.open(filename, "w") do |f|
    f << <<-EOS.gsub(/^    /, '')
    ---
    layout: post
    title: #{title}
    date: #{Time.new.strftime('%Y-%m-%d')}
    categories:
    ---

    EOS
  end

  # Automatically open the post in SublimeText
  system ("open -a MacDown #{filename}")
end

# usage: rake draft['my new draft']
desc 'create a new draft post'
task :draft, :title do |_, args|
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
    date: #{Time.new.strftime('%Y-%m-%d')}
    categories:
    ---

    EOS
  end

  # Automatically open the post in SublimeText
  system ("open -a MacDown #{filename}")
end

desc 'move a draft into the posts'
task :publish, :file do |_, args|
  file = args.file
  unless file
    files = Dir.glob("#{drafts_dir}/*.md")
    loop do
      files.each_with_index { |filename,idx| puts "#{idx+1} - #{filename}" }
      print "Post to publish: "
      choice = $stdin.gets.chomp.to_i
      file = files[choice-1] if choice > 0 && choice <= files.count
      break unless file.nil?
    end
  end
  
  file = "#{drafts_dir}/#{args.file}" unless file.start_with?("#{drafts_dir}/")
  file = "#{file}.md" unless file.end_with?('.md')

  if file && File.exist?(file)
    new_file = "#{posts_dir}/#{Time.now.strftime('%Y-%m-%d')}-#{File.basename(file)}"
    mv file, new_file
  else
    puts "File #{file} not found"
  end
end

desc 'preview the site with drafts'
task :preview do
  puts "## Generating site"
  puts "## Stop with ^C ( <CTRL>+C )"
  url = 'http://0.0.0.0:4000'
  system "(until curl -sI #{url} 1>/dev/null; do sleep 0.5; done; open #{url})&"
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
