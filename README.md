
//用了线段树哈希表实现了一个简陋的学生信息管理系统



#include<stdio.h>
#include<stdlib.h>
#include<string.h>
#define STU_NUM 101
#define ACCUR 10
#define NAMELEN 10 // Students’ names are no longer than 10 chars.
#define COURSES 4       // There are four courses in the system.
#define TRIEMAX (STU_NUM*NAMELEN)
#define HashNum (ACCUR*100)//注意大小 分开设不同变量 
void tableHead(){
    printf("ID\tName\t");
    for(int i=0;i<COURSES;++i)
        printf("CO%d\t",i);
    printf("Avg\n");
}

typedef struct stu_info {
    char stu_name[NAMELEN+10];
    int id, score[COURSES];
    float avgScore;
    struct stu_info *next;

    //默认值是很重要的
    void init(const char _stu_name[NAMELEN], const int _id,const int _score[COURSES]) {//const 
        id = _id;
        avgScore=0;//初始化 
        for (int i = 0; i < NAMELEN; ++i)
            stu_name[i] = _stu_name[i];
        for (int i = 0; i < COURSES; ++i)
            score[i] = _score[i], avgScore += (float)_score[i];
        avgScore /= COURSES;
        next=NULL;
    }

    void output() {
        printf("%d\t%s\t",id,stu_name);
        for(int i=0;i<COURSES;i++)
            printf("%d\t",score[i]);
        printf("%f\n",avgScore);
    }
} STU_INFO;

typedef struct edge {
    int to, next;//to-->id
} Edge;

struct HashTable {
    int cnt, keyNum, valueNum;
    int *head;
    Edge *edge;
    //无论是重名还是分数 边数都是对应的学生人数
    //但这个设计逻辑是不好的，hashtable应该是不知道上一级需要它做了多少的
    //点（分数）//0.0-100.0  为尾部空间进行预留

    void init(int _keyNum = HashNum, int _valueNum = STU_NUM) {//
        cnt = 0;
        keyNum = _keyNum;
        valueNum = _valueNum;
        head = (int *) malloc(sizeof(int) * _keyNum);
        edge = (Edge *) malloc(sizeof(Edge) * _valueNum);
        memset(head,-1,sizeof(int) * _keyNum);//换成堆后所有memset要重写 
        memset(edge,-1,sizeof(Edge) * _valueNum);
    }
    void destroy(){
        free(head);
        free(edge);
    }
    void add(const int u,const int v) {//c语言不支持多态 重名调用
        edge[cnt].to = v;
        edge[cnt].next = head[u];
        head[u] = cnt++;
    }
    void query(int u,STU_INFO * addr[STU_NUM]){//字典树查询时
        for (int j = head[u]; j != -1; j = edge[j].next) {
			if(edge[j].to!=-1 && addr[edge[j].to])//一旦涉及到指针和索引 判断指针是否空
                addr[edge[j].to]->output();
        }
    }
    void foreach(STU_INFO* addr[STU_NUM]) {//排序输出调用
        for (int i = 0; i < keyNum; i++)
			query(i,addr);
    }
};

struct Trie {
    HashTable hashTable;
    int  *head;
    int **next;
    int cnt, root,trieSize;

    void init(int _trieSize=TRIEMAX) {
        cnt = 1;
        root = 0;
        trieSize=_trieSize;
        hashTable.init();//相较于构造函数容易忘记初始化
        head=(int *)malloc(sizeof(int)*trieSize);
        memset(head, -1, sizeof(int)*trieSize);
        
        next=(int **)malloc(sizeof(int*)*trieSize);
        for(int i=0;i<trieSize;++i){
			next[i]=(int *)malloc(sizeof(int)*128);//ascii
			memset(next[i],-1,sizeof(int)*128);
		} 
        //memset(next, -1, sizeof(next));//就不是连续分布的内存了 
    }
    void destroy(){
        hashTable.destroy();
        free(head);
        for(int i=0;i<trieSize;++i)
            free(next[i]);
        free(next);
    }
    void insert(char str[], int id) {
        int ptr = root;
        int len = strlen(str);
        for (int i = 0; i < len; i++) {
            if (next[ptr][str[i]] == -1) {
                next[ptr][str[i]] = cnt++;
            }
            ptr = next[ptr][str[i]];
        }
        hashTable.add(ptr, id);
    }

    bool query(char str[],STU_INFO *addr[STU_NUM]) {//给底层以上层的访问权限，c语言不支持变长数组
        int ptr = root,len = strlen(str);
        for (int i = 0; i < len; i++) {
            if (next[ptr][str[i]] == -1)
                return false;
            ptr = next[ptr][str[i]];
        }
        //todo 对已删除的数据 虽然不显示，但是也无法直接返回数据已被删除
        tableHead();//输出表头
        hashTable.query(ptr,addr);
        return true;
    }

};

struct Data {
    stu_info *head;
    stu_info *cur;
    stu_info * addr[STU_NUM];//id-->addr
    Trie trie;
    HashTable hashTable;
    void init() {
        head = cur = (stu_info *) malloc(sizeof(stu_info));
        head->id=-1;
        head->next=NULL;//初始值很重要
        memset(addr, 0, sizeof(addr));
        trie.init();
        hashTable.init();
    }

    void insert(stu_info * newNode) {
        if (addr[newNode->id]) {
            printf("There is a student.\n");
            return;
        }

        cur->next = newNode;
        cur=cur->next;
        addr[newNode->id] = cur;
        trie.insert(newNode->stu_name, newNode->id);
        hashTable.add((int)(cur->avgScore * ACCUR), newNode->id);
    }

    void del(int id) {
    	if(!addr[id] || !(head->next)){
    		printf("There is not the student.\n");
            return;
		}
        addr[id] = NULL;
        stu_info* q=head;
        stu_info* ptr=head->next;
        for(;ptr!=NULL;ptr=ptr->next,q=q->next)
            if(ptr->id==id)
                break;
        q->next=ptr->next;
        free(ptr);
    }

    void output() {
        for(stu_info* ptr=head->next;ptr!=NULL;ptr=ptr->next)
            if(ptr->id!=-1)//not head todo
                ptr->output();
    }

    void sortedOutput() {
        hashTable.foreach(addr);
    }

    bool query(char stu_name[]) {
        return trie.query(stu_name,addr);
    }
    void analyze(int course,int *_max,int*_min,float* _average,float* _passingRate){
        int max=0,min=100,num=0,sum=0,score,passingNum=0;
        for(stu_info* ptr=head;ptr!=NULL;ptr=ptr->next){
            //容易忘记头节点的特殊性
            if(ptr->id!=-1){
                score=ptr->score[course];

                max=max>score?max:score;
                min=min<score?min:score;
                sum+=score;
                if(score>=60)
                    passingNum++;
                num++;
            }
        }
        *_max=max,*_min=min,*_average=(float)sum/num,*_passingRate=(float)passingNum/num;
    }
    void destroy(){
        stu_info * p=head,* q=p->next;
        while(q)//q=ptr->next;一定要确认可以访问到哦
        {
            free(p);
            p=q;
            q=q->next;
        }
        free(p);
        trie.destroy();
        hashTable.destroy();
    }
};
struct Input{
	//private
	void clear(){
		int c;
		while(( c = getchar()) != '\n' && c != EOF); 
	}
	int digit(){//不用考虑小数 
		int c,n=0,flag=0;
		while((c = getchar()) != '\n' && c != EOF){
			if(c=='-')
				flag=1;
			else if(c>='0'&& c<='9')
				n=n*10+c-'0';	
			else{
				clear();
				printf("Not Digit.Input again\n");
				n=0;
			}	
		}
		if(flag)
			return -n;
		else
			return n;
	};
	int range(int min,int max){//[min,max)
		int num=digit();
		while(num>=max||num<min){
			printf("Num illegal\n");
			num=digit();
		}
		return num;
	}
	//public
	int id(){
		return range(0,STU_NUM);
	}
	int score(){
		return range(0,100+1);
	}
	int course(){//from 1 to n
		return range(1,COURSES+1);
	}
	int op(){//from 1 to 7
		return range(1,7+1);
	}
	void name(char * stu_name){
		while(scanf("%s",stu_name) && strlen(stu_name)>NAMELEN)
			printf("Length of name is larger than expected.Please input again.\n");
		clear();
	}
}; 
struct Interface{
    Data * data;
    Input input;
    void init(Data* _data){
        data=_data;
    }
    void menu(){
        excelHead("Main Menu");
        printf("\t\t1. Show Current List\n");
        printf("\t\t2. Insert Student\n");
        printf("\t\t3. Delete Student\n");
        printf("\t\t4. Search Student\n");
        printf("\t\t5. Analyze Course\n");
        printf("\t\t6. Sort Student \n");
        printf("\t\t7. Exit the program \n");
        excelHead("");
    }
    void output(int op) {
    	if(!(data->head->next)){
    		printf("There is no student.\n");
    		return;
		}
        excelHead("List Info");
        tableHead();
        if(op==1)
            data->output();
        else
            data->sortedOutput();
        excelHead("");
    }
    void insert(){
        int id;char stu_name[NAMELEN];int score[COURSES];
        printf("ID of the new student.\n");
        id=input.id();
        printf("Name of the new student.\n");
        input.name(stu_name);
        for(int i=0;i<COURSES;++i){
            printf("Please input the score of CO%d\n",i+1);
            score[i]=input.score();
        }
        stu_info * cur=(stu_info *) malloc(sizeof(stu_info));
        cur->init(stu_name,id,score);
        data->insert(cur);
    }
    void excelHead(char * str){
        if(str=="")
            str="********";
        printf("*****************%s*****************\n",str);
    }
    void del(){
        int id;
        printf("Please enter the id of the student to delete:\n");
        printf("ID = ");
        id=input.id();
        data->del(id);
        printf("Deleter operation complete\n");
    }
    void query(){
        char stu_name[NAMELEN];
        printf("Please enter the name of the student to search:\n");
        printf("Name = \n");
        input.name(stu_name);
        if(!(data->query(stu_name)))
            printf("Student <%s> not found.\n",stu_name);
    }
    void analyze(){
        int course,max=0,min=100;
        float average,rate;
        printf("Please input the course id you want to analyze（1-n）:\n");
        course=input.course();

        data->analyze(course-1,&max,&min,&average,&rate);
		if(max==0&&min==100){
			printf("There is not course.");
			return;
		}
        printf("Course Info of CO%d :\n",course);
        printf("Max = %d\n",max);
        printf("Min = %d\n",min);
        printf("Avg = %f\n",average);
        printf("PassingRate = %f\n",rate);
    }
};
int main() {
	
    Data data;
    data.init();
    Interface interface;
    interface.init(&data);
    int op=0;
    do{
        interface.menu();
        op=interface.input.op();
        switch (op){
            case 1:
                interface.output(1);break;
            case 2:
                interface.insert();break;
            case 3:
                interface.del();break;
            case 4:
                interface.query();break;
            case 5:
                interface.analyze();break;
            case 6://sorted
                interface.output(2);break;
            case 7:
                break;
        }
    }while (op!=7);
    data.destroy();
    return 0;
    
}
