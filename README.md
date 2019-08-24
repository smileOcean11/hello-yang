#include"head.h"
char sql[BUF_SIZE]={0};
char nowUser[20]={0};
sqlite3 *db_open()
{

	sqlite3 *db=NULL;
	int ret=sqlite3_open("yang.db",&db);
	if(ret!=0)
	{
		perror("open error");
		exit(-1);
	}
	return db;
}


void db_creat()
{
	sqlite3 *db=db_open();
	sqlite3_exec(db,"create table history_record(username varchar(20),word varchar(50),time varchar(50))",NULL,NULL,NULL);
	sqlite3_exec(db,"create table user_info(username varchar(20),password varchar(20))",NULL,NULL,NULL);		
}

int user_cmp(sqlite3 *db,user_t user)
{
	memset(sql,0,sizeof(sql));
	char**result=NULL;
	int row,col;
	sprintf(sql,"select * from user_info where username='%s' and password='%s' ",user.username,user.password);
	sqlite3_get_table(db,sql,&result,&row,&col,NULL);
	if(row!=0)
	{
	return SUC;
	}
	else
	{
	puts("fail");
	return FAIL;
	}
		
}


void user_insert(sqlite3 *db,user_t user)
{
	memset(sql,0,sizeof(sql));
	sprintf(sql,"insert into user_info values('%s','%s')",user.username,user.password);
	sqlite3_exec(db,sql,NULL,NULL,NULL);
}


void history_insert(char *name,char *buf)
{	
	sqlite3 *db=db_open();
	time_t tim;
	time(&tim);
	user_t user;
	bzero(&user,sizeof(user));
	strcpy(user.username,name);
	strcpy(user.word,buf);
	sprintf(user.time,"%s",ctime(&tim));
	memset(sql,0,sizeof(sql));
	sprintf(sql,"insert into history_record values('%s','%s','%s')",user.username,user.word,user.time);
	sqlite3_exec(db,sql,NULL,NULL,NULL);
}



void history_show(int newconfd,user_t user)
{	
	
	sqlite3 *db=db_open();
	memset(sql,0,sizeof(sql));
	char**result=NULL;
	int row,col;
	int i=0;
	sprintf(sql,"select * from history_record");
	sqlite3_get_table(db,sql,&result,&row,&col,NULL);
	if(row!=0)
	{
	for(i=0;i<(row+1)*col;i++)
	{
	sprintf(user.word+strlen(user.word),"%-15s",*(result+i));
	if((i+1)%3==0)
	strcpy(user.word+strlen(user.word),"\n");
	}	
	}
	else
	{
		sprintf(user.attention,"not history");	
	}
	write(newconfd,&user,sizeof(user));
}


void register_user(int newconfd,user_t user)
{
	sqlite3 *db=db_open();
	user_insert(db,user);
	if(user_cmp(db,user)==SUC)
	{
		strcpy(user.attention,"register suc");
	}
	else
	{
		strcpy(user.attention,"register failed");
	}
	write(newconfd,&user,sizeof(user));

}


void log_user(int newconfd,user_t user)
{	
	
	sqlite3 *db=db_open();
	if(user_cmp(db,user)==SUC)
	{
		
		strcpy(nowUser,user.username);
		strcpy(user.attention,"login suc");
		
	}
	else
	{
		strcpy(user.attention,"login failed");
	}
	write(newconfd,&user,sizeof(user));
	
}

void find_word(int newconfd,user_t user)
{
	
	FILE *fp;
	fp=fopen("word.txt","r");
	if(NULL==fp)
	{
		perror("open word.txt failed");
		exit(-1);
	}
	char buf1[BUF_SIZE]={0};
	
	int fflag=1;
	while(fgets(buf1,sizeof(buf1),fp)>0)
	{
	if(strncmp(buf1,user.word,strlen(user.word))==0)
	{
		history_insert(nowUser,user.word);
		strcpy(user.word,buf1);
		fflag=0;
		break;
	}
	}
	if(fflag)
	{
	strcpy(user.attention,"not found");
	}
	else
	{
	strcpy(user.attention,"found");
	}
	write(newconfd,&user,sizeof(user));
	fclose(fp);

}


void *fun(void *p)
{
	int newconfd = *((int *)p);
	user_t user;
	while(1)
	{
	read(newconfd,&user,sizeof(user));
	switch(user.flag)
	{
	case R:register_user(newconfd,user);break;
	case L:log_user(newconfd,user);break;
	case F:find_word(newconfd,user);break;
	case H:history_show(newconfd,user);break;
	}
	}	
}


void tcp_server()
{
	int sockfd = socket(AF_INET,SOCK_STREAM,0);
	if(sockfd<0)
	{
		perror("socket error");
		exit(-1);
	}
	struct sockaddr_in server_addr;
	bzero(&server_addr,sizeof(server_addr));	
	server_addr.sin_family = AF_INET;
	server_addr.sin_port =htons(6000); 
	server_addr.sin_addr.s_addr=htonl(INADDR_ANY);

	int on=1;
	if((setsockopt(sockfd,SOL_SOCKET,SO_REUSEADDR,&on,sizeof(on)))<0)
	{
		perror("setsockopt failed");
		exit(EXIT_FAILURE);
	}


	int ret=bind(sockfd,(struct sockaddr *)&server_addr,sizeof(server_addr));
	if(ret<0)
	{
		perror("bind error");
		close(sockfd);
		exit(-1);
	}

	int ret1=listen(sockfd,5);
	if(0>ret1)
	{
		perror("listen error");
		close(sockfd);
		exit(-1);
	}
	struct sockaddr_in snd_addr;
	bzero(&snd_addr,sizeof(snd_addr));
	socklen_t len=sizeof(snd_addr);
	pthread_t tid;
	while(1)
	{
		int newfd=accept(sockfd,(struct sockaddr *)&snd_addr,&len);
		if(0>newfd)
		{
			perror("accept error");
			close(sockfd);
			close(newfd);
			exit(-1);
		}
		
		pthread_create(&tid,NULL,fun,&newfd);
		pthread_join(tid,NULL);	
	}
	close(sockfd);
	
}

int main()
{
	db_creat();
	tcp_server();
	return 0;
}
