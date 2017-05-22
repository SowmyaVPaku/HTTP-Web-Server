/***************************************************************************
 *File       : HTTPServer.c
 *Description: To create a HTTP Server in which files can be uploaded and downloaded
 *Author     : Venkata Sowmya Paku
 *Rev Date   : 26Nov2015
 *****************************************************************/


#include <stdio.h>
#include <sys/socket.h>
#include <string.h>
#include <stdlib.h>
#include <time.h>
#include <sys/stat.h>
#include <unistd.h>
#include <netinet/in.h>
#include <sys/types.h>
#include <errno.h>
#include <pthread.h>
#include <netdb.h>
#include <arpa/inet.h>
#include <fcntl.h>
#include <dirent.h>



// Define System Constraints
#define CONNMAX 					1000
#define BYTES 						1024
#define PORTNUM 					10000
#define MAX_HEADER_NAME_LENGTH      256
#define MAX_HEADER_VALUE_LENGTH     768
#define MAX_HEADER_LINE_LENGTH      1024
#define MAX_HEADERS                 64
#define MAX_METHODS                 10
#define PATH_LIMIT                    100
#define USERDBCNT					5


//Global Variables
struct client
{
	int connum;
	int Authflag;
	int head_count;
	char **Parsedmesg;
	char referer[100];
	char uploadfilename[100];
	char downfilename[100];
	char UserID[10];
	char Passwd[10];
	double requeststart;
	double requestend;
} clients[CONNMAX];

int Globalclientcount=0;

struct UserDB
{
	char UserID[10];
	char passwd[10];
}Usercredentials[USERDBCNT]={{"Dog","puppy"},{"Lion","cub"},{"U001","P001"},{"U002","P002"},{"U003","P003"}};


char *ROOT;


//Functions Declaration

void *processclient(void *clientnum);
void Parseclientmsg(char *,int );
void ProcessGet(char **Parsemesg, char *payload, int clientct);
void ProcessPost(char ** Parsemesg, char *payload, int clientct);
void Authenticate(int);
void GenDirectorylisting();
void UploadtoServer(int);
void DownloadfromServer(int);



/*****************************************************************************
 *Function    : Main
 *Description : Call initialize server and handle multiple client requests.
 *Arguments:
         argc - Number of command line parameters
         argv - Command line arguymnets passed
 *Returns: 0 on success to the OS
*****************************************************************************/
/************START***********************************FUNCTION MAIN********************************************************************/
int main(int argc,char *argv[])
{	

	int serversock;
	struct sockaddr_in clientaddr;
	socklen_t cliaddrlen;
	int pthreadclientct=0;
	
	pthread_t tid;
	
	struct timeval  tv;
    long            ms; // Milliseconds
    time_t          sec;  // Seconds
	double time_in_mill;
	
	ROOT = getenv("PWD");
	
	serversock  = initialize_server();
	printf("\nServer started at port 10000\n");
	
	// Set all elements to -1: signifies there is no client connected
    int i;
    for (i=0; i<CONNMAX; i++)
	{
        clients[i].connum=-1;
		clients[i].Authflag = -1;
		clients[i].head_count = 0;
		bzero(clients[i].uploadfilename,100);
	}
		
		
	while(1)
	{
	    cliaddrlen = sizeof(clientaddr);
        clients[Globalclientcount].connum = accept (serversock, (struct sockaddr *) &clientaddr, &cliaddrlen);
		printf("client count is %d\n", Globalclientcount);
		printf("client socket number is %d\n",clients[Globalclientcount].connum);
		if (clients[Globalclientcount].connum<0)
        perror ("accept() error");
		else
		{
			pthreadclientct = Globalclientcount;
			pthread_create(&tid,NULL,processclient,&pthreadclientct);
			Globalclientcount = (Globalclientcount+1)%CONNMAX;
		}
		
	}
	
	close(serversock);
	return 0;
}
/************END***********************************FUNCTION MAIN********************************************************************/




/*****************************************************************************
 *Function    : Initialize_server
 *Description : Initialize and bind server socket
 *Arguments: None
 *Returns: 0 on success to the OS
*****************************************************************************/
/**************************START*******************************************Intialize the server******************/
int initialize_server()
{
	int srvrsock;
	struct sockaddr_in srvraddr;
	socklen_t srvraddrlen;
	
	
	srvrsock = socket(AF_INET, SOCK_STREAM, 0); 
	
	
	//To check if the socket is created properly.
	if (srvrsock < 0) 
        perror("unable to create socket.....\n");
	
	//Clear the server before the initial connection begins.
	 bzero((char *) &srvraddr, sizeof(srvraddr)); 
	
     //Declare the server configurations.
     srvraddr.sin_family = AF_INET; 
     srvraddr.sin_addr.s_addr = INADDR_ANY;
     srvraddr.sin_port = htons(PORTNUM);
	 
	 //Bind Socket
	 if (bind(srvrsock, (struct sockaddr *) &srvraddr, sizeof(srvraddr)) < 0) 
	 {
     perror("Unable to bind the socket");
	 exit(0);
	 }
	 
	//Listen to Connections
	listen(srvrsock,CONNMAX);
	return srvrsock;
}
/**************************END*********************************************Initialize the server*********************/






/*****************************************************************************
 *Function    : Processclient
 *Description : Process each client request and capture the client message.
 *Arguments: Client counter number
 *Returns: None
*****************************************************************************/
/**************************START*******************************************Process the Clients******************/
void *processclient(void *clientct)
{
	int *clientcount = (int *)clientct;
	int n= *clientcount;
	
	printf("\n********Client ID is %d and client socket number is %d******************\n",n,clients[n].connum);
	char mesg[99999];
	int readclient;
	
	memset( (void*)mesg, (int)'\0', 99999 );
	
	readclient=read(clients[n].connum, mesg, 99999);

	
    if (readclient<0)    // receive error
        fprintf(stderr,("read() error\n"));
    else if (readclient==0)    // receive socket closed
        fprintf(stderr,"Client disconnected upexpectedly.\n");
    else    // message received
    {
		printf("%s",mesg);
		Parseclientmsg(mesg, *clientcount);
		//send(clients[n], "HTTP/1.0 404 Found\n", 23,0); //FILE NOT FOUND
	}
	bzero(mesg,99999);
	
	shutdown (clients[n].connum, SHUT_RDWR);         //All further send and recieve operations are DISABLED...
    close(clients[n].connum);
    clients[n].connum=-1; 	
}
/**************************END*******************************************Process the Clients******************/




/*****************************************************************************
 *Function    : Parseclient
 *Description : Tokenize the incoming  request and capture the client message.
 *Arguments: Message to be parsed and Client counter number
 *Returns: None
*****************************************************************************/
/**************************START*******************************************Parse Client Messages******************/
void Parseclientmsg(char * msg, int clientct)
{
	char Parsemesg[99999];
	//char   **command_line;
	char   method        [MAX_METHODS];
	char   payload_source[PATH_LIMIT];
	int    minor_version;
	int    major_version;
	char   head_name     [MAX_HEADER_LINE_LENGTH];
	char   head_value    [MAX_HEADER_VALUE_LENGTH];
	//int    head_count = 0;
	char   *token;
	int    counter=0;
	
	strcpy(Parsemesg,msg);
	clients[clientct].head_count = 0;
	//Count Number of lines in Mesg
	token = strtok (Parsemesg, "\n");
	while (token !=NULL)
	{
		clients[clientct].head_count++;
		token = strtok (NULL, "\n");
	}
	
	strcpy(Parsemesg,msg);
	
	//read first line
	
	clients[clientct].Parsedmesg= malloc( sizeof(char *)*clients[clientct].head_count);
	clients[clientct].Parsedmesg[0]=malloc(MAX_HEADER_LINE_LENGTH);
	clients[clientct].Parsedmesg[0] = strtok (Parsemesg, "\n");
	
	for (counter =1;counter <clients[clientct].head_count; counter++)
	{
		clients[clientct].Parsedmesg[counter]=malloc(MAX_HEADER_LINE_LENGTH);
		clients[clientct].Parsedmesg[counter] = strtok (NULL, "\n");
		
	}
	
	for (counter =0;counter <clients[clientct].head_count; counter++)
	{
		//printf("\n parsed string %d is %s",counter,clients[clientct].Parsedmesg[counter]);
		
		// Capture referer
		if (strncmp(clients[clientct].Parsedmesg[counter],"Referer:",8) == 0)
		{
		sscanf(clients[clientct].Parsedmesg[counter], "Referer:%s",clients[clientct].referer);		
		}
		
			// Capture Upload file name
		if (strncmp(clients[clientct].Parsedmesg[counter],"Content-Disposition: form-data; name=\"uploadedfile\"; filename=",62) == 0)
		{
		sscanf(clients[clientct].Parsedmesg[counter], "Content-Disposition: form-data; name=\"uploadedfile\"; filename=\"%s",clients[clientct].uploadfilename);		
		sscanf(clients[clientct].uploadfilename, "%s\"",clients[clientct].uploadfilename);	
		char *tok = strtok(clients[clientct].uploadfilename,"\"");
		strcpy(clients[clientct].uploadfilename, tok); 		
		}	
		
	}
	
	
// Read Header
		sscanf(clients[clientct].Parsedmesg[0], "%s %s HTTP/%d.%d%*s",method,payload_source,&major_version,&minor_version);	

	if(strcmp(method,"GET")==0)
	{
		ProcessGet(clients[clientct].Parsedmesg,payload_source,clientct);
	}
	else if(strcmp(method,"POST")==0)
	{
		ProcessPost(clients[clientct].Parsedmesg,payload_source,clientct);
	}
	
	
	
}

/**************************END*******************************************Parse Client Messages******************/







/*****************************************************************************
 *Function    : ProcessGet
 *Description : Read Parsed Message and write contents of requested page to client
 *Arguments: Parsed Message, Name of page Requeste dand Client counter number
 *Returns: None
*****************************************************************************/
/**************************START*******************************************Process GET Requests******************/
void ProcessGet(char **Parsemesg, char *payload, int clientct)
{
	int fd;
	int bytes_read;
	char data_to_send[200];
	char filepath[256];
	
	char filelist[10];
	char* tok;
	int i=0;
	
	FILE *fp;
	int nread;
	unsigned char buff[256]={0};

	if ( strncmp(payload, "/\0", 2)==0 )
	{
		strcpy(payload,"/index.html");//if no file is specified, index.html will be opened by default as in APACHE...	
	}
	
	sprintf(filepath,"%s%s",ROOT,payload);
	


//Special Cases*******************************************


		if(strncmp(payload,"/Download.html",14) == 0)
		{
				tok  = strtok (payload, "?");
			
				tok  = strtok (NULL, "=");
				
				printf("tok in download is %s\n\n\n",tok);
				
				if(tok==NULL)
				{
					strcpy(payload,"/Directory.html");
					sprintf(filepath,"%s%s",ROOT,payload);
				}
				else if(strncmp(tok,"Filename",8) == 0)
				{
					tok  = strtok (NULL, "\n");
					strcpy(clients[clientct].downfilename, tok);
					strcpy(payload,"/Download.html");
					sprintf(filepath,"%s%s",ROOT,payload);
					DownloadfromServer(clientct);
				}
		
				


		}

	
// Open requested file and send to client
	
	if ( (fd=open(filepath, O_RDONLY))!=-1 )    //FILE FOUND
    {	

		if(strncmp(clients[clientct].referer,"http://127.0.0.1:10000/Download.html",36)	 == 0)
		{
		strcpy(data_to_send,"HTTP/1.0 200 OK\n\nContent-Disposition: attachment; filename=\"fname.ext\"\n\n");
		write(clients[clientct].connum, data_to_send,strlen(data_to_send) );
		}
		else
		{
        write(clients[clientct].connum, "HTTP/1.0 200 OK\n\n", 17);
		}
        while ( (bytes_read=read(fd, data_to_send, 200))>0 )
			write (clients[clientct].connum, data_to_send, bytes_read);
		
		bzero(data_to_send, sizeof(data_to_send) );
	}
    else    
	{
		write(clients[clientct].connum, "HTTP/1.0 404 Not Found\n", 23); //FILE NOT FOUND
	}
	
}
/**************************END*******************************************Process GET Requests******************/


/*****************************************************************************
 *Function    : ProcessPost
 *Description : Read Parsed Message and write contents of requested page to client
 *Arguments: Parsed Message, Name of page Requested and Client counter number
 *Returns: None
*****************************************************************************/
/**************************START*******************************************Process POST Requests******************/
void ProcessPost(char **Parsemesg, char *payload, int clientct)
{
	//printf("Message inside POST");

	int fd;
	int bytes_read;
	char data_to_send[200];
	char filepath[512];
	int counter;



// If no file is specified default to index.html	
	if ( strncmp(payload, "/\0", 2)==0 )
	{
		strcpy(payload,"/index.html");//Because if no file is specified, index.html will be opened by default (like it happens in APACHE...	
	}


	
// Authenticate User
	Authenticate(clientct);

	
	
		
// Check Authentication Status before opening any files. If not Authenticated Yet, Only Index.html is allowed. Reset to Index.html
	if (clients[clientct].Authflag == -3 || clients[clientct].Authflag == -4)
	{
		//printf("in here");
		strcpy(payload,"/index.html");
	}
	
	bzero(filepath,sizeof(filepath));
	sprintf(filepath,"%s%s",ROOT,payload);


	if (strstr(clients[clientct].referer, ":10000/Upload.html") != NULL)
	 {
		 //printf ("TRYING TO UPLOAD FILE");
		 UploadtoServer(clientct);
	 }
 
 

//If requesting the directory contents generate the HTML file.
	if (strncmp(payload,"/Directory.html",15) ==0)
	{
//		printf("generating directory"); 
		GenDirectorylisting(); //Generate Directory.html file.
	}

// Open requested file and send to client	
	if ( (fd=open(filepath, O_RDONLY))!=-1 )    //FILE FOUND
    {		
        write(clients[clientct].connum, "HTTP/1.0 200 OK\n\n", 17);
        while ( (bytes_read=read(fd, data_to_send, 200))>0 )
			write (clients[clientct].connum, data_to_send, bytes_read);
		
		bzero(data_to_send, sizeof(data_to_send) );
	}
    else    
	{
		write(clients[clientct].connum, "HTTP/1.0 404 Not Found\n", 23); //FILE NOT FOUND
	}
	
}
/**************************END*******************************************Process POST Requests******************/	






/*****************************************************************************
 *Function    : Authenticate
 *Description : Authenticate logged in User.
 *Arguments: Client counter number
 *Returns: None
*****************************************************************************/
/**************************START*******************************************Authenticate Client******************/
void Authenticate(int clientct)
{
	int counter;
	for (counter =0;counter <clients[clientct].head_count; counter++)
	{
		if ( strncmp(clients[clientct].Parsedmesg[counter], "UserID",6)==0 )
		{
			clients[clientct].Parsedmesg[counter] = strtok(clients[clientct].Parsedmesg[counter], "=");
			clients[clientct].Parsedmesg[counter] = strtok(NULL, "&");
			strcpy(clients[clientct].UserID,clients[clientct].Parsedmesg[counter]);
			clients[clientct].Parsedmesg[counter] = strtok(NULL, "=");
			if ( strncmp(clients[clientct].Parsedmesg[counter], "password",8)==0 )
				{
						clients[clientct].Parsedmesg[counter] = strtok(NULL, "\n");
						strcpy(clients[clientct].Passwd,clients[clientct].Parsedmesg[counter]);
				}
			sscanf(clients[clientct].Parsedmesg[counter], "UserID=%s password=%s",clients[clientct].UserID,clients[clientct].Passwd);			
		}
	}
	clients[clientct].Authflag =-2; //Authentication Failed
	
	for (counter=0;counter<USERDBCNT;counter++)
	{
		if((strcmp(clients[clientct].UserID,Usercredentials[counter].UserID) == 0) && (strcmp(clients[clientct].Passwd,Usercredentials[counter].passwd) == 0))
		{
		clients[clientct].Authflag =1; //Authentication Succeeded
		break;					
		}			 
	}	
}
/**************************END*******************************************Authenticate Client******************/



/*****************************************************************************
 *Function    : GenDirectorylisting
 *Description : Generate a directory listing of files in the data folder and generate an HTML file dynamically.
 *Arguments: None.
 *Returns: None
*****************************************************************************/
/***********************START********************************Directory Listing********************************/
void GenDirectorylisting()
{
	FILE* fd = NULL;
	char filepath[512];
	char cBuffer[500];
	DIR           *d;
	struct dirent *dir;
	int counter =0;
	
	sprintf(filepath,"%s%s",ROOT,"/Directory.html");

	
	fd = fopen(filepath, "w");
	if(fd == NULL)
    {
        printf("\n fopen() Error!!!\n");
    }
	if(fd)
	{
		
		
		sprintf(cBuffer, "<html>\n<body bgcolor=\"#F0FFFF\">\n<h2 align=center>CMPE 207 File Sharepoint</h2>\n<form action=\"Download.html\" method=\"get\">\n<table table border=\"1\" align=\"center\" style=\"width:70%\"><tr><th>S.No</th><th>File Name</th><th>Download</th></tr>");
		fwrite(cBuffer, sizeof(char), strlen(cBuffer), fd) ; 


		d = opendir("./data/");
		if (d)
		{
		while ((dir = readdir(d)) != NULL)
		{
			if (strcmp(dir->d_name,".") == 0 || strcmp(dir->d_name,"..") == 0)
			{
				continue;
			}
			++counter;
			bzero(cBuffer,500);
			sprintf(cBuffer,"\n<tr><td>%d</td><td>%s</td><td><input type=\"radio\" name=\"Filename\" value=\"%s\"></td></tr>",counter,dir->d_name,dir->d_name);
			fwrite(cBuffer, sizeof(char), strlen(cBuffer), fd) ;
		}
		bzero(cBuffer,500);
		sprintf(cBuffer,"</table><br><br><div style=\"text-align: center;\"><input type=\"submit\" value=\"Download\" align=\"center\"></div>\n</form>\n<form action=\"Upload.html\" method=\"POST\"><div style=\"text-align: center;\"><input type=\"submit\" value=\"Upload File\"></div></form><div style=\"position: absolute; top: 0; right: 0; width: 100px; text-align:right;\"><form action=\"index.html\" method=\"GET\"> <input type=\"submit\" value=\"SignOut\"> </form></div></body>\n</HTML>"); 
		fwrite(cBuffer, sizeof(char), strlen(cBuffer), fd) ;
		}
	}
    closedir(d);
	fclose(fd);
	
}
/***********************END********************************Directory Listing********************************/










/*****************************************************************************
 *Function    : UploadtoServer
 *Description : Upload requested file to server.
 *Arguments: Client counter number.
 *Returns: None
*****************************************************************************/

void UploadtoServer(int clientct)
{
	FILE* fw = NULL;
	char filepath[512];
	int iCounter;
	int filestart=0;
	char buff[100000];
	

	sprintf(filepath,"%s/data/%s",ROOT,clients[clientct].uploadfilename);

	
	
	fw = fopen(filepath, "w");
	if(fw == NULL)
    {
        printf("\n fopen() Error!!!\n");
		return;
    }
	if(fw)
	{
		
		for (iCounter =0;iCounter <clients[clientct].head_count-2; iCounter++)
		{	
	
			sprintf(buff,"in Upload server string %d is %s and comparison value is %d \n", iCounter,clients[clientct].Parsedmesg[iCounter], strncmp(clients[clientct].Parsedmesg[iCounter],"\0",1));
			if  (filestart == 3)
			{
				fwrite(clients[clientct].Parsedmesg[iCounter], sizeof(char), strlen(clients[clientct].Parsedmesg[iCounter]), fw) ; 
			}
			if (filestart == 2 && strncmp(clients[clientct].Parsedmesg[iCounter],"\0",1) == 13)
			{
				filestart = 3;
			}	
			if (filestart == 1 && strncmp(clients[clientct].Parsedmesg[iCounter],"Content-Type:",13) == 0)
			{
				filestart = 2;
			}
			if (strncmp(clients[clientct].Parsedmesg[iCounter],"Content-Disposition: form-data; name=\"uploadedfile\"; filename=",62) == 0)
			{
				filestart = 1;
			}
		}			
	}
	fclose(fw);
}



/*****************************************************************************
 *Function    : DownloadfromServer
 *Description : Download requested file from server.
 *Arguments: Client counter number.
 *Returns: None
*****************************************************************************/
void DownloadfromServer(clientct)
{
	FILE* fd = NULL;
	char filepath[512];
	char cBuffer[800];
	
	sprintf(filepath,"%s%s",ROOT,"/Download.html");
	
	fd = fopen(filepath, "w");
	if(fd == NULL)
    {
        printf("\n fopen() Error!!!\n");
    }
	if(fd)
	{
		sprintf(cBuffer, "<html> \n <body bgcolor=#F0FFFF> \n <form action=\"./data/%s\" method=\"get\"> \n Downloading file %s........... <br><br><br>Click to confirm.................................................. <input type=\"submit\" value=\"Confirm\"> \n </form> \n <form action=\"Directory.html\" method=\"POST\">\n Click here to go back to Directory Listing......... \n<input type=\"submit\" value=\"Go back\">\n</form>\n <form action=\"Upload.html\" method=\"POST\"> \n Click here to Upload a file.............................. \n <input type=\"submit\" value=\"Upload File\"> \n </form>\n <div style=\"position: absolute; top: 0; right: 0; width: 100px; text-align:right;\"><form action=\"index.html\" method=\"GET\"> <input type=\"submit\" value=\"SignOut\"> </form></div></body> \n </html>", clients[clientct].downfilename,clients[clientct].downfilename);
		fwrite(cBuffer, sizeof(char), strlen(cBuffer), fd) ; 
		bzero(cBuffer,500);
	}
	fclose(fd);

}
