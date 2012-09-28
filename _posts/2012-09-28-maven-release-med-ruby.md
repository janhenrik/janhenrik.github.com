---
layout: post
title: Maven release med Ruby
---

{{ page.title }}
================

Mange av oss bruker [Maven](http://maven.apache.org) for å bygge, versjonere og holde styr på avhengigheter både til eksterne biblioteker og egne. Maven har også en [release plugin](http://maven.apache.org/plugins/maven-release-plugin/) som er Mavens forsøk på å standardisere en release-prosess, og det gjør den i tre faser over flere steg. 

* Prepare a release, [mvn release:prepare](http://maven.apache.org/plugins/maven-release-plugin/examples/prepare-release.html)
* Perform a release, [mvn release:perform](http://maven.apache.org/plugins/maven-release-plugin/examples/perform-release.html)
* Rollback a release, [mvn release:rollback](http://maven.apache.org/plugins/maven-release-plugin/rollback-mojo.html)

Innføring av denne i vårt prosjekt bød på flere problemer. Prosjektet vårt har en del andre krav enn det Maven release-pluginen forventer. Vi bruker git istedet for subversion, vi har standardiserte måter å versjonere komponenter på, slik at neste versjon alltid kan utledes av forrige. I tillegg synes vi release-prosessen til Maven brukte evigheter på å analysere, bygge og starte opp. Derfor forsøkte vi å skrive dette selv, med utgangspunkt i Maven sin dokumentasjon. Prosjektet hadde kompetanse på Ruby, og vi følte dette var et godt utgangspunkt for å parse XML og enkelt kunne kjøre git-kommandoer. Følgende er vår løsning på stegene beskrevet i Maven sin release-prosess:

#### mvn release:prepare: 1. Check that there are no uncommitted changes in the sources
Dette er lett å implementere. En oneliner med git.
	def clean_wd?
	  `git status --porcelain -- . | wc -l`.strip == "0"
	end

#### mvn release:prepare: 2. Check that there are no SNAPSHOT dependencies
Vi bruker [Nokogiri](http://nokogiri.org) for å parse XML, Ruby trenger ikke å kjenne til Mavens dependency-tre for de enkle tingene release-prossen skal gjøre. I og med at det er veldefinerte steder i POM-filene SNAPSHOT-dependencies kan være, var dette trivielt.

	def snapshot_dependencies?
	    self.search('dependency').each do |dep|
	      dep.search('version').each do |ver|
	        if ver.content =~ /SNAPSHOT/i
	          raise("Prosjektet #{artifactId} har SNAPSHOT avhengigheter
	               i artifact: #{artifactIdForDep(dep)}-#{ver.content}")
	        end
	      end
	    end

	# Tilsvarende for <properties> og project > parent > version  # for super-pom

#### mvn release:prepare: 3. Change the version in the POMs from x-SNAPSHOT to a new version (you will be prompted for the versions to use)
Vi har veldefinerte regler for bumping av versjonsnummer. Alltid minor-versjon, evt. patchversjon dersom det er spesifisert på kommandolinjen inn til scriptet.

	def neste_snapshot release_version
	  versionsnummer = release_version.split('.')
	  major = versionsnummer[0].to_i
	  minor = versionsnummer[1].to_i
	  patch = versionsnummer[2].to_i

	  if patch == 0
	    return "#{major}.#{minor+1}.0"
	  else
	    return "#{major}.#{minor}.#{patch+1}"
	  end
	end

#### mvn release:prepare: 4. Transform the SCM information in the POM to include the final destination of the tag
Et unødvendig steg, for å være helt ærlig skjønner vi ikke hva som menes. Dersom vi ikke forstår det, bør vi mest sannsynlig heller ikke implementere noe.

#### mvn release:prepare: 5. Run the project tests against the modified POMs to confirm everything is in working order
Dette er eneste gangen vi bygger prosjektet og sjekker at alle tester kjører.

	mvn_success = system("mvn -U clean package >> /tmp/mvn_build")
	  unless mvn_success
	    raise("Build failure")
	  end

#### mvn release:prepare: 6. Commit the modified POMs
Commit av release-versjonen til git.

	`git commit pom.xml */pom.xml -m "[release.rb] Release ny versjon av
	                  #{super_pom.artifactId}-#{release_version}"`


#### mvn release:prepare: 7. Tag the code in the SCM with a version name (this will be prompted for)
Tag av kode for release-versjonen.

	unless system("git tag -f #{super_pom.artifactId}-#{release_version}")
	    raise("Could not create tag.")
	  end

#### mvn release:prepare: 8. Bump the version in the POMs to a new value y-SNAPSHOT (these values will also be prompted for)
Igjen er det en veldefinert måte å bumpe versjonsnummer på, denne gangen til neste utviklingsversjon.

	def neste_release version
	  versionsnummer = version.split('.')
	  major = versionsnummer[0].to_i
	  minor = versionsnummer[1].to_i
	  return "#{major}.#{minor+1}.0-SNAPSHOT"
	end

#### mvn release:prepare: 9. Commit the modified POMs
En commit av neste utviklingsversjon, og her kommer til slutt også push til sentralt repo. 

	puts "Sjekker inn ny utviklingsversjon"
	  `git commit pom.xml */pom.xml -m 
	  	"[release.rb] Ny utviklingsversjon av 
	  		#{super_pom.artifactId}-#{snapshot_version}"`
	puts "Pusher til sentralt git repository."
	  `git push --tags && git push origin #{current_branch}`

Ved å pushe til sentralt repo helt til slutt får vi en atomisk operasjon, slik at evt. rollback blir enkelt. 

#### mvn release:release: 1. Checkout from an SCM URL with optional tag
Nå som prepare-steget er ferdig kan releasen-fasen starte. Maven release-pluginen laster ned koden igjen fra tag og bygger. Dette er ikke nødvendig hvis du har distribuert versjonskontroll, da kan du eventuelt få konflikt ved push til master til slutt, men selve bygget blir likt det vi gjorde i steget over. Derfor bygger vi ikke igjen.

#### mvn release:release: 2. Run the predefined Maven goals to release the project (by default, deploy site-deploy)
Vår release-prossess laster opp til artifactory. Dersom denne feiler er det viktig at det gjøres igjen, da releaset kode er pushet til master. Derfor har vi også parametrisert scriptet, slik at denne operasjonen kan gjøres uavhengig av "prepare-steget".

	  printf "Laster opp #{local_file} til #{rest_of_path}..."
	  begin
	    res = RestClient.put(url, File.new(local_file))
	    unless res.code >= 200 and res.code < 300
	      raise "Opplasting til Artifactory svarte ikke med HTTP OK, 
	      	feilmelding var: #{res.message}"
	    end
	    puts "Opplasting vellykket! Status: #{res.code}"
	  rescue => e

#### mvn release:rollback
Ved feil før eller under push til sentralt git-repo, må vi resette lokal historie i git. Det blir vårt rollback-steg og utføres manuelt.

	git reset --hard origin/HEAD

## Konklusjon
Vi bruker Maven kun til bygg, versjonering og avhengigheter. Der fungerer det bra.

Vårt script for release er på ~200 linjer Ruby-kode, og langt over dobbelt så raskt som Maven release-plugin og inneholder kun det vi forstår og våre egne forretningsregler for versjonering etc. Koden er enkel å forstå og endre på. Slike prosesser er bedre egnet for script enn Javakode og blir mer synlig på denne måten.


