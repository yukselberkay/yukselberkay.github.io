---
layout: post
title:  "[EN] How to Write a Key-Value Database with Rust"
categories: programming
---
# Table of Contents
- [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Why With Rust ?](#why-with-rust-)
  - [Use cases](#use-cases)
  - [Frontend](#frontend)
  - [Backend](#backend)
  - [Further Improving Our Database With Indexing.](#further-improving-our-database-with-indexing)
  - [Usage](#usage)
  - [Summary](#summary)

## Introduction
While I am working on my pet password manager project ([pmanager](https://github.com/yukselberkay/pmanager/tree/dev)) a problem is quickly arised where I needed to store username and password pairs in a reliable manner. I could not make any assumptions about where I exactly store and retrieve data. For instance I can't say something like "this and that entries will always be stored and retrieved in this place in the file". This approach would not be dynamic. I could use a SQL database and it would make a lot of sense because with it, I could retrieve and store username and password pairs regardless of where it is in the database. For instance I can write a query like "SELECT * FROM ENTRIES WHERE domain='twitter.com'" and this would bring all username and password pairs regarding "twitter.com". But this would introduce a whole new dependency to the system like PostgreSQL and would make the project more complex than I like. I could use an encrypted JSON file and retrieve username password pairs like that but it would not ensure the integrity of the data inside. So I set out to write my own key value database which would support the features that I needed (Get, List, Insert, Update, Delete) while ensuring that the data inside is not corrupted. Which turned out to be more fun to implement than other alternatives, I ended up using an encrypted and slightly modified version of this database for my project. (Which will be released soon.)

![](/assets/keyvaldb.png)

## Why With Rust ?
While I was working on another pet project of mine to teach myself about the inner workings of a shell program by implementing one (with C) ([minsh](https://github.com/yukselberkay/minsh)) I was quickly introduced with the concept of memory leaks. After I ran my program I noticed an unnecessary amount of memory usage after investigating this issue using valgrind (checks memory allocation and de-allocation routines.) I noticed that I was not de-allocating some of the memory I was using. While debugging and mitigating this issue was a good learning experience, I did not want give too much attention to the problems that are not directly related to the problem that I want to solve.

Many programming languages solves this issue with a runtime environment. Java uses a garbage collector which tracks memory and free's the parts of the memory which no longer used by the program. Which is a solid and field tested approach but this operation is done on runtime which comes with an extra performance penalty.

Rust solves this issue rather cleverly. Rust's compiler ensures that every allocated memory is dropped when its no longer used. It does this with an ownership system. It calls drop implicitly when a related data goes out of scope. The advantage of this apporach is that there is no runtime penalty anymore. So it runs fast as C while ensuring memory safety.

Memory leak introduces some security implications than just using unnecessary system resources like denial of service. If for example users can leak memory with an input they give, they can make the program to use even more memory. Which could interrupt the services running on the same system. For more information -> <https://owasp.org/www-community/vulnerabilities/Memory_leak>


Some pros and cons about Rust:

Pros:
- Performance and efficient memory usage.
- Concurrency is easier and safer.
- A great dependency management and build system.

Cons:

- Compilation times are long.
- Development times can be slower while getting started. Compiler can be hard to satisfy sometimes.

## Use cases
I want my key value database to:

- "Get" a specified entry.
- "List" every entry.
- "Insert" an entry.
- "Update" an existing entry.
- "Delete" an existing entry.

Lets start coding.

## Frontend
This is the part that interacts with the backend of our database. It gets the path for our database file and pass that information to the backend (Which I will explaing in the following section.) 

We use "match" to parse the arguments that we need. It allows us to handle every possible case explicitly, so we dont miss any possibility that can come with "&args.command".

```rust
/*
* main.rs
* Code that interacts with our backend.
*/
mod args;

use crate::args::Subcommands;
use libkvdb::KeyValueDB;

use std::path::{Path};

fn main() {
    let args = args::arg_parser();

    let fname = args.path;
    let db_path = Path::new(&fname);

    // opens the database file at path
    let mut store = KeyValueDB::open(db_path).expect("unable to open database file");
    // loads the offsets of any pre-existing data into an in-memory index.
    store.load().expect("unable to load data from database");

    match &args.command {
        Some(Subcommands::Get { key }) => {
            println!("Get {}", key);
            let result = store.get(key.as_bytes()).unwrap().unwrap();
            println!("{:?}", result);
        },
        Some(Subcommands::Insert { key, value }) => {
            println!("Insert {} -> {}", key, value);
            store.insert(key.as_bytes(), value.as_bytes()).unwrap();
        },
        Some(Subcommands::Delete { key }) => {
            println!("Delete -> {}", key);
        },
        Some(Subcommands::Update { key, value }) => {
            println!("Update -> {}, {}", key, value);
        },
        Some(Subcommands::List {  }) => {
            dbg!("list subcommand supplied");
            store.list();
        }
        // prints out generated help message automatically
        None => {}
    }
}
```

## Backend
The first two things that we do is opening our database for reading and writing and loading the offsets of any pre existing data into an in-memory index. (Populating "index" hashmap) What the loading part essentially does is that it matches the data with its position in the file and it inserts it into a hashmap which is the most important part of our db.
Opening
```rust
 pub fn open(path: &Path) -> io::Result<Self> {
        let f = OpenOptions::new()
            .read(true)
            .write(true)
            .create(true)
            .append(true)
            .open(path)?;
    
        let index: HashMap<ByteString, u64> = HashMap::new();
        Ok(KeyValueDB{ f, index })
    }
```
Loading the contents to memory.
```rust
 pub fn load(&mut self) -> io::Result<()>  {
        let mut f = BufReader::new(&mut self.f);

        // this loop reads db file and maps the positions of records to index hashmap.
        // then we can use these positions to reach our values.
        loop {
            // returns the number of the bytes from the start of the file.
            // this becomes the value of the index.
            let position = f.seek(SeekFrom::Current(0))?;

            dbg!(&position);

            // read a record in the file at its current position
            let maybe_kv = KeyValueDB::process_record(&mut f);

            let kv = match maybe_kv {
                Ok(kv) => {
                    //dbg!(&kv);
                    kv
                },
                Err(err) => {
                    match err.kind() {
                        io::ErrorKind::UnexpectedEof => {
                            break;
                        }
                        _ => return Err(err)
                    }
                }
            };

            self.index.insert(kv.key, position);
            //dbg!(&self.index);
        }

        Ok(())
    }
```

To load this index into memory, we need to process individual records. A record consists of the checksum of the data, length of the key and length of the value. Then we proceed to allocate just enough data to store this one record. After that we check the integrity of the data by comparing stored checksum and calculating the checksum data in the memory. If these two does not match, it means that the data is corrupted somehow (maybe some other program arbitrarily wrote data to it.) and we panic. We can call "checksum", "key_len" and "val_len" our header information. We read or write our data by processing this header information.
Byteorder crate is used to write data in a deterministic manner, free from specific architecture. So this piece of code can run on little endian systems as well as big endian systems.

```rust
    fn process_record<R: Read> (f: &mut R) -> io::Result<KeyValuePair> {
        // the reason that we use byteorder crate here is that
        // we want to write data to our disk in a deterministic way
        // free from the platform we are on.
        let saved_checksum = f.read_u32::<LittleEndian>()?;
        let key_len = f.read_u32::<LittleEndian>()?;
        let val_len = f.read_u32::<LittleEndian>()?;
        let data_len = key_len + val_len;

        let mut data = ByteString::with_capacity(data_len as usize);

        {
            f.by_ref().take(data_len as u64)
                .read_to_end(&mut data)?;
        }

        debug_assert_eq!(data.len(), data_len as usize);

        let checksum = crc32::checksum_ieee(&data);
        // ensure data integrity
        if checksum != saved_checksum {
            panic!("data corruption");
        }
        let value = data.split_off(key_len as usize);
        let key = data;
        Ok(KeyValuePair{key, value})
    }
```

After these operations, doing the operations that we want to do is a matter of manipulating data from our in memory index (our hashmap data structure). To list or get data we lookup in our hashmap, and if there is a match we return it.
```rust
    pub fn list(
        &mut self,
    ) {
        for (key, val) in &self.index {
            let mut f = BufReader::new(&mut self.f);
            // seek  
            f.seek(SeekFrom::Start(*val)).unwrap();
            let kv: KeyValuePair = KeyValueDB::process_record(&mut f).unwrap();
   

            let s_key = String::from_utf8_lossy(key);
            let s_val = String::from_utf8_lossy(&kv.value);

            println!("key -> {:?} pos -> {:?}", s_key, s_val);
        }
    }

    pub fn get(
        &mut self,
        key: &ByteStr
    ) -> io::Result<Option<ByteString>> {
        let position = match self.index.get(key) {
            None => return Ok(None),
            Some(position) => *position,
        };

        let kv = self.get_at(position)?;

        Ok(Some(kv.value))
    }

    pub fn get_at(
        &mut self,
        position: u64
    ) -> io::Result<KeyValuePair> {
        let mut f = BufReader::new(&mut self.f);
        f.seek(SeekFrom::Start(position))?;
        let kv = KeyValueDB::process_record(&mut f)?;

        Ok(kv)
    }
```

To insert data, we calculate the header data that we need to produce an individual record. We calculate checksum, key length and value length. We determine where we will insert the data with seek and then we use insert operation on our hashmap. This is an append only database so every insert operation will be written at the end of the file.
```rust
pub fn insert(&mut self,
        key: &ByteStr,
        value: &ByteStr
    ) -> io::Result<()> {
        let position = self.insert_but_ignore_index(key, value)?;

        self.index.insert(key.to_vec(), position);
        Ok(())
    }

    pub fn insert_but_ignore_index(
        &mut self,
        key: &ByteStr,
        value: &ByteStr,
    ) -> io::Result<u64> {
        let mut f = BufWriter::new(&mut self.f);

        let key_len = key.len();
        let val_len = value.len();
        let mut tmp = ByteString::with_capacity(key_len + val_len);

        for byte in key {
            tmp.push(*byte);
        }
        for byte in value {
            tmp.push(*byte);
        }

        let checksum = crc32::checksum_ieee(&tmp);

        let next_byte = SeekFrom::End(0);
        let current_position = f.seek(SeekFrom::Current(0))?;
        f.seek(next_byte)?;
        f.write_u32::<LittleEndian>(checksum)?;
        f.write_u32::<LittleEndian>(key_len as u32)?;
        f.write_u32::<LittleEndian>(val_len as u32)?;
        f.write_all(&mut tmp)?;

        Ok(current_position)
    }
```

Implementing insert gives us "update" and "delete" operations for free.
```rust
    pub fn update(
        &mut self,
        key: &ByteStr,
        value: &ByteStr,
    ) -> io::Result<()> {
        self.insert(key, value)
    }

    pub fn delete(
        &mut self,
        key: &ByteStr
    ) -> io::Result<()> {
        self.insert(key, b"")
    }
```

## Further Improving Our Database With Indexing.
The problem with this approach is that everytime that we want to get some operation done we load this in memory index by parsing our database file. The solution to this problem is indexing. If we store the position of the entries inside our database to a separate file, and reading this file instead of loading our database for every operation this would give us a huge performance boost when storing a large amount of data inside.

## Usage
You can build and run this code by cloning my repository with the instructions given in its README -> <https://github.com/yukselberkay/KeyValDb>

## Summary

Check out the progression of my password manager project which includes an encrypted and slightly modified version of this database. -> <https://github.com/yukselberkay/pmanager/tree/dev> 

Thanks to Tim McNamara for Rust In Action book. Which helped me a lot to understand and implement these concepts and to improve it further. 

Let me know if you have any questions and if I made any mistake in this post.

Thank you for reading.