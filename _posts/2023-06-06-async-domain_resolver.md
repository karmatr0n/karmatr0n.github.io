---
layout: post
title: "Asynchronous Domain Resolver in Rust"
date:  2023-06-06 18:47:00
tags:  domain networking dns resolver rust async 
---

Rust application to resolve domains asynchronously using DNS queries with the A and AAAA DNS record types 
to identify if the domain is resolving or not.

There same operations can be performed with tools like [massdns](https://github.com/blechschmidt/massdns)
or bash scripts using the dig or host commands, but I wanted to learn how to do a basic version of those tools
using Rust.

Example to look up for the example.com domain:
{% highlight bash linenos %}

dig +short example.com
93.184.216.34


dig +short asdfasdf.io # No IP address found
{% endhighlight %}

The two DNS queries to identify if the domain is resolved are based on the following [DNS Record 
types](https://en.wikipedia.org/wiki/List_of_DNS_record_types):

* A: IPv4 address
* AAAA: IPv6 address

The code is available on [github](https://github.com/karmatr0n/domain_resolver).

The application is composed of following parts:

* Domain: Stores the domain and its resolved status.
* DomainNames: Stores the resolved domains and serialize them to JSON.
* DomainGenerator: Reads names from a file and combine them with the top level domain to build a list of domain names.
* DomainResolver: Resolves the domain names asynchronously in batches of 20 and save the results into a file.

The following sequence diagram shows the components and how they interact with each other.
![Sequence Diagram](/img/domain_resolver/sequence_diagram.png)

## Domain
The Domain struct defines two fields: name and resolved. Additionally, this also provides a 
new function to create a Domain instance and uses the Serialize and Deserialize traits 
from the [serde crate](https://serde.rs/).

{% highlight rust linenos %}
#[derive(Serialize, Deserialize)]
struct Domain {
    name: String,
    resolved: bool,
}

impl Domain {
      fn new(name: String, resolved: bool) -> Self {
          Self {
              name: name,
              resolved: resolved,
          }
      }
  }

{% endhighlight %}

## DomainNames
The DomainNames struct has a single field called domains of type Vec<Domain>, and provides 
methods to create a new instance, add domains to the list, and convert this to JSON format.
{% highlight rust linenos %}
struct DomainNames {
    domains: Vec<Domain>,
}

impl DomainNames {
    fn new() -> Self {
        Self {
            domains: Vec::<Domain>::new(),
        }
    }

    fn add(&mut self, domain: Domain) {
        self.domains.push(domain);
    }

    fn to_json(&mut self) -> Result<String, serde_json::Error> {
        serde_json::to_string_pretty(&self.domains)
    }
}
{% endhighlight %}

## DomainGenerator
This struct has methods for creating a new instance as well as generating valid domains from a 
file with names combined with the provided top level domain. As result, this returns a vector 
storing the generated domains.

{% highlight rust linenos %}
impl DomainGenerator {
    fn new(path: String, top_level: String) -> Self {
        Self {
            path: path,
            top_level: top_level,
        }
    }

    fn generate_domains(&self) -> io::Result<Vec<String>> {
        let file_path = Path::new(&self.path);
        let file = File::open(&file_path)?;
        self.words_to_domains(BufReader::new(file))
    }

    fn words_to_domains<R: BufRead>(&self, reader: R) -> io::Result<Vec<String>> {
        let mut domains = Vec::new();
        for line in reader.lines() {
            let name = line?;
            if name.is_empty() {
                continue;
            }
            let domain = format!("{}.{}", name, self.top_level);
            if self.valid_domain(&domain) {
                domains.push(domain);
            }
        }
        Ok(domains)
    }

    fn valid_domain(&self, domain: &String) -> bool {
        let regex_pattern = r"^[a-zA-Z0-9][a-zA-Z0-9-]{1,61}[a-zA-Z0-9]\.[a-zA-Z]{2,}$";
        let regex = Regex::new(regex_pattern).unwrap();
        regex.is_match(domain)
    }
}
{% endhighlight %}

## AsyncDomainResolver
This Rust code defines a struct used for asynchronously resolving multiple domain names.  It has fields for a 
list of domains, the maximum number of concurrent lookups, and uses the data structure called 
DomainNames to store the resolved domains.  The provided methods  allow to perform the following actions:
* 
* Initializing a new instance of AsyncDomainResolver. 
* Resolving domains using asynchronous operations in batches of 20.
* Returns the results as a JSON string.

{% highlight rust linenos %}
struct AsyncDomainResolver {
    domains: Vec<String>,
    max_async_lookups: u32,
    resolved_domains: DomainNames,
}

impl AsyncDomainResolver {
    fn new(domains: Vec<String>) -> Self {
        Self {
            domains: domains,
            max_async_lookups: 20,
            resolved_domains: DomainNames::new(),
        }
    }

    fn resolve_domains(&mut self) {
        let rt = tokio::runtime::Runtime::new().unwrap();
        for domains in self.domains.chunks(self.max_async_lookups as usize) {
            let now = Local::now();
            println!("--- Verifying {} domains at {:?} ---", domains.len(), now);
            let verified_domains = rt.block_on(self.async_resolve_domains(domains));
            for domain in verified_domains {
                println!("domain: {}, resolved:{}", domain.name, domain.resolved);
                self.resolved_domains.add(domain);
            }
        }
    }

    fn as_json(&mut self) -> Result<String, serde_json::Error> {
        self.resolved_domains.to_json()
    }

    async fn async_resolve_domains(&self, domains: &[String]) -> Vec<Domain> {
        let mut futures = Vec::new();
        for domain in domains {
            let f = self.resolve_domain(domain.to_string());
            futures.push(f);
        }
        let results = join_all(futures).await;
        results
    }

    async fn resolve_domain(&self, qname: String) -> Domain {
        let ip_addr_and_port = "8.8.8.8:53";
        let nameserver: SocketAddr = ip_addr_and_port
            .parse()
            .expect("Unable to parse socket address");

        let config = ClientConfig::with_nameserver(nameserver);
        let mut client = Client::new(config)
            .await
            .expect("Unable to create DNS client");

        let rrset = client.query_rrset::<A>(qname.as_str(), Class::In).await;
        let rrset_ipv6 = client.query_rrset::<Aaaa>(qname.as_str(), Class::In).await;
        Domain::new(qname, rrset.is_ok() || rrset_ipv6.is_ok())
    }
}
{% endhighlight %}

## Main
The main function defines a command-line interface using the clap crate. It expects two or three arguments: 

* Path of file with names 
* Top level domain of your choice
* Output path (optional)

The main function parses the command-line arguments, initializes a DomainGenerator, generates domains, 
resolves them asynchronously using AsyncDomainResolver, then converts the results to JSON, 
and writes it to the specified output file. 

{% highlight rust linenos %}
fn main() {
    let args = Args::parse();

    let generator = DomainGenerator::new(args.names_path, args.top_level);

    match generator.generate_domains() {
        Ok(domains) => {
            let mut async_resolver = AsyncDomainResolver::new(domains);
            async_resolver.resolve_domains();
            if let Ok(json) = async_resolver.as_json() {
                let output_path = args.output_path;
                fs::write(output_path.clone(), json).expect("Unable to write file");
                println!("Output file: {}", output_path);
            } else if let Err(e) = async_resolver.as_json() {
                eprintln!("Failed to convert to JSON: {}", e);
            }
        }
        Err(e) => eprintln!("Error occurred: {}", e),
    }
}
{% endhighlight %}

## How to get the code
{% highlight bash linenos %}
git clone https://github.com/karmatr0n/domain_resolver.git
{% endhighlight %}

## How to compile the application
{% highlight bash linenos %}
cargo build --release
{% endhighlight %}

## How to install it
{% highlight bash linenos %}
cargo install --path .
{% endhighlight %}

## How to run it
{% highlight bash linenos %}
$HOME/.cargo/domain_resolver -n dicts/wordlist -t io -o results.json

or

$HOME/.cargo/domain_resolver -n dicts/wordlist -t io 
{% endhighlight %}

## Example output
{% highlight json linenos %}
[
  {
    "name": "qwerty.io",
    "resolved": true
  },
  {
    "name": "iloveu.io",
    "resolved": true
  },
  {
    "name": "michelle.io",
    "resolved": true
  },
  {
    "name": "tigger.io",
    "resolved": true
  },
  {
    "name": "sunshinexabc.io",
    "resolved": false
  },
  {
    "name": "chocolate.io",
    "resolved": true
  },
  {
    "name": "password1.io",
    "resolved": false
  }
]
{% endhighlight %}

## Conclusion

This article demonstrates how to resolve domains asynchronously using Rust, and it has been one of my favorite recent 
personal projects so far. Writing this small application has taught me about asynchronous programming in Rust, 
which is one of the concurrent programming models supported by this language. It also allow me to combine this 
knowledge with my interest in network protocols.

I hope you enjoyed reading it and learned something new.

## References
* [rust-lang](https://www.rust-lang.org/) for the Rust programming language.
* [Asynchronous Programming in Rust](https://rust-lang.github.io/async-book/) for asynchronous programming in Rust.
* [clap](https://crates.io/crates/clap) for the command-line interface.
* [tokio](https://crates.io/crates/tokio) for the asynchronous runtime.
* [rsdns](https://crates.io/crates/rsdns) for the Asynchronous DNS client.
* [chrono](https://crates.io/crates/chrono) for working with date and time.
* [serde](https://crates.io/crates/serde) for serializing the domain names to JSON.



