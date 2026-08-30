# Linux Notes

## Variables

## File Systems
- All the data that stored on computer is organized into files and directories
- Files and directories are organized into tree structure called file system
- Directories are like containers that holds the files or other directories
- Files are dump of raw binary data 0's and 1's. the bytes in the file can represent text, images, videos, audio etc...
- Filepath is the text representation of the location/path to the current working directory

## Absolute and Relative Paths
- Relative file paths are the paths relative to the current working directory. Where as the Absolute file path is the exact path from root to the directory.
- Absolute file paths always works from anywhere in the system, as the eventually trace back to the root directory and go to the path.
- Relative file paths depends on where we are currently.
- Use Absolute paths for more explicit, use relative paths for the directories inside the current directory

## Files
- Files are the blobs of data. The raw bytes in a file can represent anything like text, audio, video, images etc..
- To view the content of a file we can use the **cat** command like ```cat file1.txt```
- cat represents concatenate we can combine the contents of one or more file like ```cat file1.txt file2.txt```

## Head and Tail
- Sometimes the contents of files can be very very large, so that we cannot print them ( like log files )
- In these scenarios we can use the head and tail commands.
- **head** command prints first n lines of a file like ```head -n 5 file1.txt```
- **tail** command prints last n lines of a file like ```head -n 5 file1.txt```

## More and Less
- These commands lets us view the contents of a file, one page / line at a time.
- **less** command does everything that **more** command does but also has some more features.
- So we should use less command instead of more command.
- Use more command only when the system doesn't have less command installed.

## Touch
- ***touch*** command updates the access and modification timestamps of a file.
- If the specified file doesn't exists *touch* will automatically create an empty file with the same name.
- Command usage ```touch file.txt```
- We can also create multiple files like ```touch file1.txt file2.txt```
- *touch* ensures files exist without altering the existing ones, only creating new files if necessary.

## Directories
- Directories are the locations in the file system, that contains files and other directories.
- In order to create a new directory inside the current directory we can use the command ```mkdir <directory-name>```

## Move
- *move* command moves a file/directory from one location to another location.
- We can use this command to either rename a file or move it to another directory all together.
- While moving the file the destination directory should not be same sa the source directory.
- Renaming a file ```mv file1.txt renamed-file1.txt```
- Moving a file ```mv file1.txt directory2/file1.txt```

## Remove
- *remove* command delete a file or empty a directory.
- To remove a file :- ```rm file1.txt```
- If we want to delete a directory we need to use ```rm -r directory```. It deletes the directory and its contents recursively

## Copy
- To copy the contents from one location to another location, we can use copy command.
- Command :- ```cp source_file.txt destination/```
- To copy whole directory and it's contents recursively we can use the following command :- ```cp -R source_dir dest_dir```

## Alias ( ~ )
- ~ character is an alias for home directory.

## Grep
- *grep* command allows us to search for text in files.
- Command :- ```grep "hello" file1.txt```
- It prints every line in file1.txt that contains word hello.
- grep is a case sensitive search
- Inorder to search in multiple files ```grep "hello" file1.txt file2.txt```
- We can also perform search across a directory including all of it's subdirectories using ```grep -r "hello" .```


## Find
- *find* command is useful for finding files and directories by name, not by their contents.
- To find a file-by-name we can use :- ```find directory-name -name hello.txt```
- We can also search for the files that match a pattern using a find command :- ```find directory-name -name "*.txt"```
    - finds all the files that ends with .txt
- * is a wildcard character that matches anything.


## Users
- Unix-like systems supports multiple users. Each user can have their own home-directory, own files and own permissions.


## Sudo
- *sudo* command let's us run a command as superuser.
- sudo = superuser do
- We need a password to run commands using sudo
- sudo is very critical command, as it can hamper a lot of system things.


## Permissions
- In a unix-like system, permissions control who can do what to which files and directories.
- The permissions of an individual file or directory are usually represented as a 10 character string 

## Changing Permissions
- *chmod* command let's us change the permissions of a file or directory
