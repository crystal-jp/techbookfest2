AUTHORS=%w(5t111111 at_grandpa arcage tbrand makenowjust)
BUILDS=AUTHORS.map {|a| "#{a}/README.txt" }

task :build => BUILDS do
  sh "echo '<SJIS-MAC>' > build/all.txt"
  BUILDS.each do |b|
    sh "cat #{b} | nkf -s >> build/all.txt"
  end
end

task :clean do
  rm(BUILDS)
end

task :tags => BUILDS do
  paras = []
  chars = []

  BUILDS.each do |b|
    doc = File.read(b)
    doc.scan(/<ParaStyle:([^>]+)>/) do |m|
      paras << m[0]
    end
    doc.scan(/<CharStyle:([^>]+)>/) do |m|
      chars << m[0]
    end
  end

  puts "ParaStyle ==>"
  puts paras.uniq
  puts "CharStyle ==>"
  puts chars.uniq
end

AUTHORS.each do |author|
  file "#{author}/README.txt" => ["#{author}/README.md"] do |t|
    sh "md2inao.pl --format=in_design #{t.source} | sed -e '1d' | sed -e 's/ParaStyle:タイトル>タイトル：/ParaStyle:タイトル>/' > #{t.name}"
  end
end
