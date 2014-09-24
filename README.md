#define _SVID_SOURCE
#include <stdio.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <string.h>
#include <signal.h>
#include <setjmp.h>
#include <stdlib.h>
#include <fcntl.h>
#include <ctype.h>
#include <unistd.h>
#define MAX_CHAR 1024 // Max characters for Input and array sizes

char PROMPT[MAX_CHAR];
char PATH[MAX_CHAR];
char HOME[MAX_CHAR];
char last_dir[MAX_CHAR];
char message[MAX_CHAR];
char alias_for_cd[MAX_CHAR];
int cd_has_alias = 0;

/* Structure for alias */

typedef struct {
	char *akey;
	char *val;
} alias;

int num_of_alias;
alias **aliases;

jmp_buf (getinput); /* Saving environment for setjmp and longjmp */

/* Setting home directory for new shell*/

int setHomeDirectory(char home[]) {
	int chk;
	chk = chdir(home);
	if (chk != 0) {
		printf("\nError Setting home directory\n");
		return 0;
	}
	return 1;
}


/* Reads PROFILE file and sets environment variables */

void setEnv() {
	FILE *infile = fopen("profile.txt", "r");
	char line[MAX_CHAR];
	char word[MAX_CHAR];
	int i = 0;
	int j = 0;
	do {
		fscanf(infile, "%c", &word[i]);
	} while (word[i] != ':');
	do {
		fscanf(infile, "%c", &word[i]);
		message[j++] = word[i];
	} while (word[i] != '\n');
	message[j - 1] = '\0';
	j = 0;
	do {
		fscanf(infile, "%c", &word[i]);
	} while (word[i] != ':');
	do {
		fscanf(infile, "%c", &word[i]);
		PROMPT[j++] = word[i];
	} while (word[i] != '\n');
	PROMPT[j - 1] = '\0';
	j = 0;
	do {
		fscanf(infile, "%c", &word[i]);
	} while (word[i] != ':');
	do {
		fscanf(infile, "%c", &word[i]);
		PATH[j++] = word[i];
	} while (word[i] != '\n');
	PATH[j - 1] = '\0';
	j = 0;
	do {
		fscanf(infile, "%c", &word[i]);
	} while (word[i] != ':');
	do {
		fscanf(infile, "%c", &word[i]);
		HOME[j++] = word[i];
	} while (word[i] != '\n');
	HOME[j - 1] = '\0';

	int x=setHomeDirectory(HOME);
	if (x != 1) {
		printf("\nError Setting home directory\n");
	}
}


/* Function to spawn new process */

int spawnProcess() {
	int pid;
	pid = fork();
	int flag = 0;
	if (pid == 0) {
		exit(0);
		flag = 1;
	}
	printf("The new process id is %d \n", pid);
	return flag;
}

/* Function to parse the input command */

void parse(char *line, char**argv) {
	while (*line != '\0') {
		while (*line == ' ' || *line == '\t' || *line == '\n')
			*line++ = '\0';
		*argv++ = line;
		while (*line != '\0' && *line != ' ' && *line != '\t' && *line != '\n')
			line++;
	}
	*argv = '\0';
}


/* Executing commands with pipe | in them */

int executePipeCommand(char* line) {
	FILE *pipein_fp, *pipeout_fp;
	int i, j;
	char readbuf[80];
	char* command1;
	char* command2;
	int flag = 1;
	
	command1 = strtok(line, "|");
	command2 = strtok(NULL, "|");
	i = 0;
	while (command2[i] == ' ') {
		j = 0;
		while (command2[j]) {
			command2[j] = command2[j + 1];
			j++;
		}
		i++;
	}
	if ((pipein_fp = popen(command1, "r")) == NULL) {
		printf("Error executing command\n");
		perror("popen");
	    longjmp(getinput, 1);
	}
	if ((pipeout_fp = popen(command2, "w")) == NULL) {
		printf("Error execting command\n");
		perror("popen");
		longjmp(getinput, 1);
		flag = 1;
	}
	while (fgets(readbuf, 80, pipein_fp)) {
	flag=0;
		fputs(readbuf, pipeout_fp);
	}
	pclose(pipein_fp);
	pclose(pipeout_fp);
	return flag;
}


/* Executing all cases of cd command */

int executeCdCommand(char command[]) {
	int chk;
	char dir[100];
	char last_dir_temp[MAX_CHAR];
	int i = 0, j = 0;
	while ((command[i] != ' ') && (command[i] != '\0'))
		i++;
	if (command[i] == ' ') {
		i++;
		while (command[i] != '\0') {
			dir[j++] = command[i++];
		}
		dir[j] = '\0';

		if (dir[0] == '-') {
			getcwd(last_dir_temp, MAX_CHAR);
			chdir(last_dir);
			strcpy(last_dir, last_dir_temp);
		} else {
			getcwd(last_dir_temp, MAX_CHAR);
			chk = chdir(dir);
			if (chk != 0) {
				printf("\nError Setting directory\n");
				return 0;
			}
			strcpy(last_dir, last_dir_temp);
			return 1;
		}
	} else if (command[i] != ' ') {
		getcwd(last_dir, MAX_CHAR);
		setHomeDirectory(HOME);
		return 1;
	}
	return 0;
}

/* Function to execute a command to redirect output to file */

int executeRedirectCommand(char* line) {
	FILE *pipein_fp, *redirect_fp;
	int redirect;
	char readbuf[128];
	int i, j;
	char* command1;
	char* command2;
	int pid, fd;
	int flag = 1;
	command1 = strtok(line, ">");
	command2 = strtok(NULL, ">");
	while (command2[i] == ' ') {
		j = 0;
		while (command2[j]) {
			command2[j] = command2[j + 1];
			j++;
		}
		i++;
	}
	redirect_fp = fopen(command2, "w");
	if ((pipein_fp = popen(command1, "r")) == NULL) {
		longjmp(getinput, 1);
		}
	while (fgets(readbuf, 80, pipein_fp)) {
	flag=0;
		fputs(readbuf, redirect_fp);
		fflush(redirect_fp);
	}
	fclose(redirect_fp);
	pclose(pipein_fp);
	return flag;
}

/* Function to execute command without arguments, pipe or redirect */

int executeCommand(char* line) {
	FILE *pipein_fp;
	char readbuf[80];
	int flag = 1;
	if ((pipein_fp = popen(line, "r")) == NULL) {
		printf("Error executing command\n");
		perror("popen");
		longjmp(getinput, 1);
	}
	while (fgets(readbuf, 80, pipein_fp))
	{
		flag=0;
		printf("%s", readbuf);
	}
	//printf("\nFLAG is %d\n",flag);
	pclose(pipein_fp);
	return flag;
}

/* Bulding alias when alias command is given */
// enter command lika alias l ls. you can use it like l or alias m mkdir. you can use it like m dir1
void buildAlias(char* line) {

	char *key;
	char *val;

	char delims[] = " ";
	char *result = NULL;
	int i = -1;
	result = strtok(line, delims);
	while (result != NULL) {

		i++;
		if (i == 1) {
			key = malloc(strlen(result) + 1);
			if(key==NULL)
			{
			printf("Error: Out of memory!!\n");
			exit(1);
			}
			strcpy(key, result);
		} else if (i == 2) {

			val = malloc(strlen(result) + 1);
			if(val==NULL)
			{
			printf("Error: Out of memory!!\n");
			exit(1);
			}
			strcpy(val, result);

		}
		result = strtok(NULL, delims);

	}
		int j=-1;
	for(j=0;j<num_of_alias;j++){
	if((strcmp(key,aliases[j]->akey)==0)||(strcmp(val,aliases[j]->val)==0))
	{
	printf("\n Error: Alias already exists!!\n");
	break;
	}
	}
	if(j==num_of_alias)
	{
	aliases[num_of_alias] = (alias *) malloc(sizeof(alias));
	if(aliases[num_of_alias]==NULL)
			{
			printf("Error: Out of memory!!\n");
			exit(1);
			}
	aliases[num_of_alias]->akey = key;
	aliases[num_of_alias]->val = val;
	num_of_alias++;
	}
	key = NULL;
	val = NULL;

}

/* This function is used to replace the alias with the original command */
char *replace(char *st, char *orig, char *repl) {

	char delims[] = " ";
	char *result = NULL;

	result = strtok(st, delims);
	if((result = strtok(NULL, delims))==NULL){
		return repl;
	}else{
		char *fstring = malloc(strlen(repl)+2);
	if(fstring==NULL)
			{
			printf("Error: Out of memory!!\n");
			exit(1);
			}
		strcpy(fstring,repl);
		while (result != NULL) {

			strcat(fstring," ");
            char *t = malloc(strlen(fstring)+1);
			if(t==NULL)
			{
			printf("Error: Out of memory!!\n");
			exit(1);
			}
            strcpy(t,fstring);


            int newln = strlen(fstring)+strlen(result);
            fstring = realloc(fstring, newln+1);
            strcpy(fstring, t);
            strcat(fstring,result);
			result = strtok(NULL, delims);
		}
		return fstring;
	}


}

/* Function to find which command the alias represents */

char *getCommandForAlias(char *cmd) {

	int j = -1;
	for (j = 0; j < num_of_alias; j++) {
		if (strcmp(aliases[j]->akey, cmd) == 0) {
			free(cmd);
			char *val = aliases[j]->val;
			return val;
		}
	}
	return cmd;
}

/* Function to process the alias command */

char *processAlias(char *param_input){


	if (strstr(param_input, ">") != NULL)
		return param_input;
	else if (strstr(param_input, "cd") != NULL)
		  return param_input;
	else if (strstr(param_input, "if") != NULL)
		  return param_input;
	else if (strstr(param_input, "|") != NULL)
		return param_input;
	else{
		char *input = malloc(strlen(param_input)+1);
		if(input==NULL)
			{
			printf("Error: Out of memory!!\n");
			exit(1);
			}
	    strcpy(input,param_input);
		char delims[] = " ";
		char *alias = NULL;
	    alias = strtok(input, delims);
		
        char *realCommand = getCommandForAlias(alias);
		
		char *modifiedinput = replace(param_input,alias,realCommand);
		return modifiedinput;
	}
}

/* Signal handler for CTRL+C */

void ctrlc_handler(int c) {
	char* answer;
	signal(SIGINT, SIG_IGN);
	do {
		printf("\nAre you sure you want to exit? (Y/N)");
		fflush(stdin);
		answer = fgets(answer, MAX_CHAR, stdin);
		if (*answer == 'y' || *answer == 'Y') {
			printf("\nShutting down shell...\n");
			exit(0);
		} else if (*answer == 'n' || *answer == 'N') {
			signal(SIGINT, ctrlc_handler);
			longjmp(getinput, 1);
		} else
			printf("\n\nChoice not valid (Y/N)!\n\n");
	} while (*answer != 'y' && *answer != 'Y' && *answer != 'n'
			&& *answer != 'N');
}


/* Function to parse and execute If command */
void executeIfCommand(char line[]){
	char command1[1024];
    char command2[1024];
	char command3[1024];
	char line1[1024];
	char readbuf[80];
	int i, j;
	int flag;
	FILE *pipein_fp, *pipeout_fp, *pipeelse_fp;
	
	
	if(line[0] == 'i' && line[1] == 'f'){
		i = 3;
		j = 0;
		while(1){
			if(line[i] == 't' && line[i+1] == 'h' && line[i+2] == 'e' && line[i+3] == 'n')
				break;
			command1[j++] = line[i++];
		}
		i = i+4;
		command1[j] = '\0';
		//printf("command1 : %s\n", command1);
		j=0;
		while(1){
			if(line[i] == 'e' && line[i+1] == 'l' && line[i+2] == 's' && line[i+3] == 'e')
			break;
			command2[j++] = line[i++];
		}
		command2[j] = '\0';
		//printf("command2 : %s\n", command2);
		i = i+4;
		j=0;
		while(line[i] != '\0'){
			if(line[i] == 'f' && line[i+1] == 'i')
				break;
			command3[j++] = line[i++];
		}
		command3[j] = '\0';
		//printf("command3 : %s\n", command3);
	}
	
	i=0;
	while(command1[i] == ' ')
    {
		j = 0;
		while(command1[j])
		{
			command1[j] = command1[j+1];
			j++;
		}
		i++;
	}
	//printf("command1 : %s\n",command1);

	i = 0;
	while(command2[i] == ' ')
	{
		j = 0;
		while(command2[j])
		{
			command2[j] = command2[j+1];
			j++;
		}
		i++;
	}
	//printf("command2 : %s\n",command2);

	i = 0;
	while(command3[i] == ' ')
	{
		j = 0;
		while(command3[j])
		{
			command3[j] = command3[j+1];
			j++;
		}
		i++;
	}
	//printf("command3 : %s\n",command3);
	
	
	if(strstr(command1,">")!=NULL)
	{
		flag = executeRedirectCommand(command1);
	}else if (strcmp(command1,"getpid")==0)
		flag = spawnProcess();
	else if ((command1[0]=='c')&&(command1[1]=='d')&& ((command1[2]==' ')||(command1[2]=='\0')))
	{
		flag = executeCdCommand(command1);
	}
	else if (strstr(command1,"|")!=NULL)
		flag = executePipeCommand(command1); 
	else 
		flag = executeCommand(command1);
		
	if (flag != 1)
	{
		printf("\nCommand in IF executes \n");
		printf("\nExecuting THEN \n");
		if(strstr(command2,">")!=NULL)
		{
			executeRedirectCommand(command2);
		}else if (strcmp(command2,"getpid")==0)
			spawnProcess();
		else if ((command2[0]=='c')&&(command2[1]=='d')&& ((command2[2]==' ')||(command2[2]=='\0')))
		{
			executeCdCommand(command2);
		}
		else if (strstr(command2,"|")!=NULL)
			executePipeCommand(command2);
		else
			executeCommand(command2);

		
    }else {
		printf("\n Command in IF Failed\n");
		printf("\n Executing ELSE \n");
		if(strstr(command3,">")!=NULL)
		{
			executeRedirectCommand(command3);
		}else if (strcmp(command3,"getpid")==0)
			spawnProcess();
		else if ((command3[0]=='c')&&(command3[1]=='d')&& ((command3[2]==' ')||(command3[2]=='\0')))
		{
			executeCdCommand(command3);
		}
		else if (strstr(command3,"|")!=NULL)
			executePipeCommand(command3); 
		else
			executeCommand(command3);

		}
}

/* Start point of the new shell, Main function */

int main(int argc, const char* argvs[]) {

	num_of_alias = 0;

	char line[1024], line1[1024];
	int i;
	char *temp_line;
	char *argv[64];
	int len = 0;
	signal(SIGINT, ctrlc_handler);
	printf("\nWELCOME TO MY NEW SHELL!!\n\n");
	printf("Project Group 2\n\n");
	printf("Manju Muralidharan Priya\n");
	printf("Harika Thadakamalla\n");
	printf("Swasthi Tripathy\n");
	
	setEnv();
	printf("\n\n%s\n\n",message);
	setjmp(getinput);

	while (1) {

		printf("%s", PROMPT);
		gets(line);
		if (line[0] != '\0') {
			strcpy(line1, line);
			fflush(stdin);
			parse(line, argv);

			if (strcmp(argv[0], "exit") == 0) {
				printf("\n Shell is shutting down....\n");
				
				exit(0);
			}
			
			/* special case for alias for cd */
			if (cd_has_alias == 1)
            {
                len = strlen(alias_for_cd);
                if ((strncmp(line1,alias_for_cd,len) == 0) &&
                   ((line1[len] == ' ') || (line1[len] == '\0')))
                {
                    executeCdCommand(line1);
					/* continue with while(1) loop for processing next command */ 
					continue;
                }                
            }
			
			char *modifiedinput = processAlias(line1);
			
			if(strstr(line1, "if") != NULL) {
				executeIfCommand(line1);
			}
			else 			
			if (strstr(modifiedinput, ">") != NULL) {
				executeRedirectCommand(modifiedinput);
			} else if (strcmp(modifiedinput, "getpid") == 0)
				spawnProcess();			
			else if (strstr(modifiedinput, "|") != NULL)
				executePipeCommand(modifiedinput);
			else if ((line1[0]=='c')&&(line1[1]=='d')&&((line1[2]==' ')||(line1[2]=='\0')))
            {
                executeCdCommand(line1);
            }
			else if (strstr(modifiedinput, "alias") != NULL)
			{	
                len = strlen(line1);
                if ((line1[len-2] == 'c') && (line1[len-1] == 'd'))
                {
				    if (cd_has_alias == 1)
                    {
                        printf("\nError: Alias already exists!!\n");
                    }
                    else
                    {
					    /* format:  alias c cd */
                        if ((len <= 9) || (line1[5] != ' ') || (line1[6] == ' '))
                        {
                            printf("\nIncorrect format for alias\n");
                        }
                        else
                        {
                            i = 6;
                            while ((i < len-3) && (line1[i] != ' '))
                            {
                                alias_for_cd[i-6] = line1[i];
                                i++;
                            }
                            alias_for_cd[i] = '\0';
                            cd_has_alias = 1;
                        }
                    } 
                }
                else
                {  				    
					buildAlias(modifiedinput);
				}	 
			}			
			else
				executeCommand(modifiedinput);
		}
	}

	return 1;
}

