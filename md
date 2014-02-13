#!/usr/bin/env node

var fs         = require( 'fs' ),
	exec       = require('child_process').exec,
	argumentum = require( 'argumentum' ),
	prompt     = require( 'cli-prompt' ),
	colours    = require( 'colors' ),
	queue      = require( 'func-queue' );

var dir_projects = __dirname + '/projects/';

var config = {
	script: 'md',
	commandRequired: true,
	
	commands: {
		'projects': {
			help: 'Display all the available projects.',
			callback: function( options ) {
				H.projects_list( 'Below are all the available projects, enter one to view its configuration:' );
				
				if ( !H.projects().length ) return false;
				prompt.multi([
					{ key: 'name' , label: 'View configuration for project (ctrl+c to cancel)'.grey , validate: V.project }
				], function( input ) {
					var project = require( H.project( input.name ) );
					
					console.log( project );
				});
			}
		},
		'project-create': {
			help: 'Create a new project configuration file.',
			callback: function( options ) {
				prompt.multi([
					{ key: 'name' , label: 'Project name (pretty) *'.grey , validate: V.req },
					{ key: 'slug' , label: 'Project slug (sluggified) *'.grey , validate: V.slug },
					
					{ key: 'ssh_name' , label: 'Please enter the server name configured in ~/.ssh/config *'.grey , validate: V.req },
					
					{ key: 'remote_host' , label: 'Remote database host *'.grey , validate: V.req },
					{ key: 'remote_user' , label: 'Remote database username *'.grey , validate: V.req },
					{ key: 'remote_pass' , label: 'Remote database password'.grey },
					{ key: 'remote_db'   , label: 'Remote database name *'.grey , validate: V.req },
					
					{ key: 'local_host' , label: 'Local database host *'.grey , default: 'localhost' },
					{ key: 'local_user' , label: 'Local database username *'.grey , default: 'root' },
					{ key: 'local_pass' , label: 'Local database password'.grey },
					{ key: 'local_db'   , label: 'Local database name *'.grey , validate: V.req }
				], function( input ) {
					var string = JSON.stringify( input , null , 4 );
					
					fs.writeFile( H.project( input.slug ) , string , function( err ) {
						if ( err ) {
							console.log( 'Oops... something went wrong!'.red );
							console.log( err.red );
						}
						else {
							console.log( '%s project configuration has been saved successfully.'.green , input.name );
						}
					});
				});
			}
		},
		'project-delete': {
			help: 'Remove an existing project configuration file.',
			callback: function( options ) {
				H.projects_list( 'Enter one of the following available projects below to remove:' );
				
				prompt.multi([
					{ key: 'name' , label: 'Project slug *'.grey , validate: V.project },
					{ key: 'confirm' , label: 'Are you sure?'.yellow , type: 'boolean' }
				], function( input ) {
					var project = H.project( input.name );
					
					if ( !input.confirm ) return false;
					
					fs.unlink( project );
					console.log( 'Removed project (%s) successfully.'.green , input.name );
				});
			}
		},
		
		'db-backup': {
			help: 'Create a database backup for a specific project.',
			callback: function( options ) {
				H.projects_list( 'Enter one of the following available projects to perform a database backup on:' );
				
				prompt.multi([
					{ key: 'name' , label: 'Project slug *'.grey , validate: V.project },
					{ key: 'output' , label: 'Outout file (no extension) *'.grey , validate: V.req }
				], function( input ) {
					var project = require( H.project( input.name ) ),
						command = [ 'mysqldump' ];
					
					// Append the file extension for the output file.
					input.output += '.sql';
					
					command.push( '-u' , project.local_user );
					command.push( '-h' , project.local_host );
					
					// Add a password if provided. Password param and password should have no space between them.
					if ( project.local_pass ) {
						command.push( '-p' + project.local_pass );
					}
					
					command.push( project.local_db , '>' , input.output );
					
					exec( command.join( ' ' ) , function( error , stdout , stderr ) {
						if ( stderr ) {
							console.log( stderr.red );
							return false;
						}
						
						console.log( 'Database dump successfully created at %s'.green , input.output );
					});
				});
			}
		},
		'db-restore': {
			help: 'Restore a database dump for a specific project.',
			callback: function( options ) {
				H.projects_list( 'Enter one of the following available projects to perform a database backup on:' );
				
				prompt.multi([
					{ key: 'name' , label: 'Project slug *'.grey , validate: V.project },
					{ key: 'input' , label: 'Input file path *'.grey , validate: V.req }
				], function( input ) {
					var project = require( H.project( input.name ) ),
						command = [ 'mysql' ];
					
					if ( !fs.existsSync( input.input ) ) {
						console.log( 'Database dump (%s) does not exist.'.red , input.input );
						return false;
					}
					
					command.push( '-u' , project.local_user );
					command.push( '-h' , project.local_host );
					
					// Add a password if provided. Password param and password should have no space between them.
					if ( project.local_pass ) {
						command.push( '-p' + project.local_pass );
					}
					
					command.push( project.local_db , '<' , input.input );
					
					exec( command.join( ' ' ) , function( error , stdout , stderr ) {
						if ( stderr ) {
							console.log( stderr.red );
							return false;
						}
						
						console.log( 'Database dump successfully restored from %s'.green , input.input );
					});
				});
			}
		},
		
		'db-pull': {
			help: 'Pull a database dump from a server and apply it locally.',
			callback: function( options ) {
				H.projects_list( 'Enter one of the following available projects to perform a database restore from a remote server:' );
				
				prompt.multi([
					{ key: 'name' , label: 'Project slug *'.grey , validate: V.project }
				], function( input ) {
					var project   = require( H.project( input.name ) ),
						dump_file = project.slug + '_dump.sql';
					
					// Create a queue to we can run our tasks synchronously.
					var q = new queue.createQueue(function( error , code ) {
						console.log( error.red.bold );
					}, function() {
						console.log( 'All done.'.green.bold );
					});
					
					var execute_command = function( command, msg ) {
						process.stdout.write( (msg + '... ').grey );
						exec( command.join( ' ' ), function( error , stdout , stderr ) {
							if ( stderr ) return q.error( stderr );
							
							console.log( 'done.'.green );
							q.deliver();
						});
					};
					
					q.add(function() {
						var command = [ 'ssh' , project.ssh_name , '"' , 'mysqldump' ];
						
						command.push( '-u' , project.remote_user );
						command.push( '-h' , project.remote_host );
						
						// Add a password if provided. Password param and password should have no space between them.
						if ( project.remote_pass ) {
							command.push( '-p' + project.remote_pass );
						}
						
						command.push( project.remote_db , '>' , dump_file );
						command.push( '"' );
						
						execute_command( command , 'Creating remote database dump' );
					});
					q.add(function() {
						var command = [ 'scp' ];
						
						command.push( project.ssh_name + ':~/' + dump_file );
						command.push( '.' );
						
						execute_command( command , 'Downloading dump file' );
					});
					q.add(function() {
						var command = [ 'mysql' ];
						
						command.push( '-u' , project.local_user );
						command.push( '-h' , project.local_host );
						
						// Add a password if provided. Password param and password should have no space between them.
						if ( project.local_pass ) {
							command.push( '-p' + project.local_pass );
						}
						
						command.push( project.local_db , '<' , dump_file );
						
						execute_command( command , 'Applying dump file to local database' );
					});
					q.add(function() {
						var command = [ 'rm' , dump_file , '&&' ];
						
						command.push( 'ssh' , project.ssh_name , '"' , 'rm' , dump_file , '"' );
						
						execute_command( command , 'Cleaning up' );
					});
					
					q.execute();
				});
			}
		},
		'db-push': {
			help: 'Push a local database dump apply it to a remote server.',
			callback: function( options ) {
				H.projects_list( 'Enter one of the following available projects to perform a local database dump to a remote server:' );
				
				prompt.multi([
					{ key: 'name' , label: 'Project slug *'.grey , validate: V.project }
				], function( input ) {
					var project   = require( H.project( input.name ) ),
						dump_file = project.slug + '_dump.sql';
					
					// Create a queue to we can run our tasks synchronously.
					var q = new queue.createQueue(function( error , code ) {
						console.log( error.red.bold );
					}, function() {
						console.log( 'All done.'.green.bold );
					});
					
					var execute_command = function( command, msg ) {
						process.stdout.write( (msg + '... ').grey );
						exec( command.join( ' ' ), function( error , stdout , stderr ) {
							if ( stderr ) {
								return q.error( stderr );
							}
							
							console.log( 'done.'.green );
							q.deliver();
						});
					};
					
					q.add(function() {
						var command = [ 'mysqldump' ];
						
						command.push( '-u' , project.local_user );
						command.push( '-h' , project.local_host );
						
						// Add a password if provided. Password param and password should have no space between them.
						if ( project.local_pass ) {
							command.push( '-p' + project.local_pass );
						}
						
						command.push( project.local_db , '>' , dump_file );
						
						execute_command( command , 'Creating local database dump' );
					});
					q.add(function() {
						var command = [ 'scp' , dump_file ];
						
						command.push( project.ssh_name + ':~/' );
						
						execute_command( command , 'Uploading dump file' );
					});
					q.add(function() {
						var command = [ 'ssh' , project.ssh_name , '"' , 'mysql' ];
						
						command.push( '-u' , project.remote_user );
						command.push( '-h' , project.remote_host );
						
						// Add a password if provided. Password param and password should have no space between them.
						if ( project.remote_pass ) {
							command.push( '-p' + project.remote_pass );
						}
						
						command.push( project.remote_db , '<' , dump_file );
						command.push( '"' );
						
						execute_command( command , 'Applying database dump to remote server' );
					});
					q.add(function() {
						var command = [ 'rm' , dump_file , '&&' ];
						
						command.push( 'ssh' , project.ssh_name , '"' , 'rm' , dump_file , '"' );
						
						execute_command( command , 'Cleaning up' );
					});
					
					q.execute();
				});
			}
		}
	}
};

// Simple validations to use on the prompt method.
var V = {
	req: function( val ) { if ( val.length < 1 ) throw new Error( 'Value must be at least 1 characters long.'.red ); },
	slug: function( val ) {
		V.req( val );
		if ( ! /^[a-z_]+$/.test( val ) ) throw new Error( 'Value must be all lowercase characters and underscores.'.red );
		if ( fs.existsSync( H.project( val ) ) ) throw new Error( 'A project with the same slug (%s) already exists'.red , val );
	},
	project: function( val ) {
		V.req( val );
		if ( !fs.existsSync( H.project( val ) ) ) throw new Error( 'The project you entered does not exist. Please enter a project from the list above.'.red );
	}
};

// Simple helper functions.
var H = {
	project: function( val ) { return dir_projects + val + '.json'; },
	projects: function( removeExt ) {
		var projects = fs.readdirSync( dir_projects ),
			json     = [];
		
		for ( var i in projects ) {
			if ( projects[ i ].indexOf( '.json' ) >= 0 ) {
				if ( removeExt === true ) {
					projects[ i ] = projects[ i ].replace( '.json' , '' );
				}
				json.push( projects[ i ] );
			}
		}
		
		return json;
	},
	projects_list: function( msg ) {
		var projects = H.projects( true );
		
		console.log( msg.grey );
		console.log( projects.join( ', ' ).blue );
	}
};

argumentum.load( config ).parse();