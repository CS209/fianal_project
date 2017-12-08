---
title: Function.py
notebook: Function.ipynb
nav_include: 6

---


```python
import pandas as pd
import numpy as np
import math
import copy
import datetime
import time
import requests
import pandas as pd
from bs4 import BeautifulSoup
import os.path
import re
import matplotlib.pyplot as plt
import seaborn as sns
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import KFold
from scipy.interpolate import interp1d
import statsmodels.api as sm

# Plot Settings For all Notebooks Uniformity
plt.rcParams['figure.facecolor']='w'
plt.rcParams['axes.labelsize']=15
plt.rcParams['xtick.labelsize']=15
plt.rcParams['ytick.labelsize']=15

class NBAWinProbability:
	def __init__(self,team_stats = True, Seasons=range(2013,2018),FolderName='Data/'):
		self.team_stats = team_stats
		self.DataFloder=FolderName
		self.AllSeasons= Seasons
	#### BEGINING INTERNAL FUNCTIONS ######
	def __GetLinks(self,Month,Season):
		url = 'https://www.basketball-reference.com/leagues/NBA_{}_games-{}.html'.format(Season,Month)
		response = requests.get(url)

		soup = BeautifulSoup(response.text, 'lxml')    
		links = []
		for ref in soup.find_all('a'):
			link = ref.get('href')
			if link.startswith('/boxscores/2'):
				links.append(link)

		urls = []
		for link in links:
			link=link.split('/')
			string='https://www.basketball-reference.com' + ''.join(('/boxscores/pbp/',link[-1]))
			urls.append(string)
		return urls
	
	def __CreateCSV(self, url):
		response = requests.get(url)
		soup = BeautifulSoup(response.text, 'lxml')

		title = soup.find_all('title')[0].text
		pbp = pd.read_html(str(soup),header=1)[0]
		awayteam = title.replace(' Play-By-Play',' at ').split(' at ')[0]
		hometeam = title.replace(' Play-By-Play',' at ').split(' at ')[1]
		keys=pbp.keys()
		pbp=pbp.rename(columns={keys[1]:awayteam,keys[5]:hometeam})
		CSVName='PBPRecord_'+url.split('/')[-1].split('.')[0]+'.csv'
		pbp.to_csv(self.DataFloder+CSVName)

	def __CreateNameCSV(self, allLinks):
		lists=[]
		for url in allLinks:
			CSVName='PBPRecord_'+url.split('/')[-1].split('.')[0]+'.csv'
			lists.append([CSVName,url])
		pd.DataFrame(lists).to_csv(self.DataFloder+'AllFilesList.csv')

	def __CreatePBPFeatures(self, CVSFile):
		FSL = len(self.DataFloder)
		year = int(CVSFile[FSL+10:FSL+14])
		date = int(CVSFile[FSL+14:FSL+18])
		game_name = CVSFile[FSL+10:FSL+22]
		print(game_name)
		if date<800:
			season = year
		else:
			season = year + 1
		pbp = pd.read_csv(CVSFile)
		pbp = pbp.drop(['Unnamed: 0'], axis=1)
		pbp.at[0,'Score'] = '0-0'
		pbp.at[1,'Score'] = '0-0'
		pbp.at[0,'Time'] = 720

		awayteam = pbp.columns[1]
		hometeam = pbp.columns[5]
		pbp.columns = ['time', 'awayevents','awaypts','score','homepts','homeevents']
		pbp['awayteam'] = awayteam
		pbp['hometeam'] = hometeam
		events = pbp['awayevents']
		events = events.fillna(pbp['homeevents'])
		pbp['event'] = events
		pbp['isawayevent'] = 1-pd.isnull(pbp['awayevents'])
		pbp['isawaypossess'] = np.zeros(len(pbp))
		for k in range(len(pbp)-1):
			pbp.at[k,'isawaypossess'] = pbp.at[k+1,'isawayevent']

		#first need to replace scores at the beginning of games and the rows that just mark the end of quarters 
		pbp['score'] = pbp['score'].replace(to_replace='Score',method='ffill')
		pbp['score'] = pbp['score'].fillna(method='bfill')

		pbp = pbp.dropna(axis = 0, subset = ['score'])
		score = [x.split('-') for x in pbp['score']]
		awayscore,homescore = np.transpose(np.array(score))
		awayscore = [int(x) for x in awayscore]
		homescore = [int(x) for x in homescore]
		pbp['awayscore'] = awayscore
		pbp['homescore'] = homescore

		#now drop the redundant variables
		pbp = pbp.drop(['awayevents','awaypts','score','homepts','homeevents'], axis=1)

		new_columns = ["Defensive rebound", "Offensive rebound", "makes free throw", "makes technical free throw", 
					   "makes 2-pt shot", "makes 3-pt shot", 
					   "misses free throw", "misses technical free throw", "misses 2-pt shot", "misses 3-pt shot", 
					   "Personal foul", "Personal take foul", "Loose ball foul", "Shooting foul", "Technical foul", "Turnover", 
					   "full timeout", "assist by"]
		countaway_columns = []
		counthome_columns = []
		for columns in new_columns:
			countaway_columns.append('count away '+columns)
			counthome_columns.append('count home '+columns)

		pbp1 = copy.deepcopy(pbp)
		pbp1 = pbp1.dropna()
		pbp1["quarter"] = np.zeros(len(pbp1))
		pbp1["quarter_real"] = np.zeros(len(pbp1))
		pbp1.at[0,'quarter'] = 1; pbp1.at[0,'quarter_real'] = 1
		for column in new_columns:
			pbp1[column] = np.zeros(len(pbp1))
		for column in countaway_columns:
			pbp1[column] = np.zeros(len(pbp1))
		for column in counthome_columns:
			pbp1[column] = np.zeros(len(pbp1))

		quarter = 1; quarter_real = 1;
		fulltimeout_time = 720
		# fulltimeout_count = [6,6]
		index = pbp1.index
		for k in range(1,len(pbp1)):
			# modify time left to seconds
			time_string = pbp1.iloc[k]['time']
			if time_string[-2] == '.':
				x = time.strptime(time_string.split('.')[0],'%M:%S')
				time_second = datetime.timedelta(minutes=x.tm_min,seconds=x.tm_sec).total_seconds()
			pbp1.at[index[k],'time'] = time_second

			event = pbp1.iloc[k]['event']
			# extract quarter
			if "Start" in event and "quarter" in event:
				quarter = int(event[9])
				quarter_real = int(event[9])
			if "Start" in event and "overtime" in event:
				print(event)
				quarter = 4               
				quarter_real = int(event[9]) + 4
			pbp1.at[index[k],'quarter'] = quarter
			pbp1.at[index[k],'quarter_real'] = quarter_real
			# check events
			for col in range(len(new_columns)):
				column = new_columns[col]
				pbp1.at[index[k],countaway_columns[col]] = pbp1.at[index[k-1],countaway_columns[col]]
				pbp1.at[index[k],counthome_columns[col]] = pbp1.at[index[k-1],counthome_columns[col]]
				# if the event happens
				if column in event:
					pbp1.at[index[k],column] = 1
					if pbp1.iloc[k]['isawayevent'] == 1:
						pbp1.at[index[k],countaway_columns[col]] = pbp1.at[index[k-1],countaway_columns[col]] + 1
					else:
						pbp1.at[index[k],counthome_columns[col]] = pbp1.at[index[k-1],counthome_columns[col]] + 1

		pbp1['time'] = pbp1['time'] + 720*(4-pbp1['quarter'])
		for k in range(0,len(pbp1)):
			if pbp1.iloc[k]['quarter_real'] < 5: # not in overtime
				pbp1.at[index[k], 'real time'] = pbp1.iloc[k]['time'] + (quarter_real-4)*300 #number of overtimes
			else: # inside overtime
				
				pbp1.at[index[k], 'real time'] = pbp1.iloc[k]['time'] + (quarter_real- pbp1.iloc[k]['quarter_real'])*300
		pbp1 = pbp1.drop(['quarter'], axis = 1)
		
		###########################################
		tshome = pd.read_csv('team_stats/'+str(season)+'home.csv')
		tsaway = pd.read_csv('team_stats/'+str(season)+'away.csv')
		tshome['WR'] = tshome['W']/tshome['GP']
		tsaway['WR'] = tsaway['W']/tsaway['GP']
		tshome = tshome.drop(['GP','W','L','MIN'], axis=1);
		tsaway = tsaway.drop(['GP','W','L','MIN'], axis=1);

		# standarize the continuous attributes
		for column in tshome.columns:
			if column != 'TEAM':
				mean=np.mean(tshome[column])
				std=np.std(tshome[column])
				tshome[column]=tshome[column].map(lambda x: (x - mean)/std)
		for column in tshome.columns:
			if column != 'TEAM':
				mean=np.mean(tsaway[column])
				std=np.std(tsaway[column])
				tsaway[column]=tsaway[column].map(lambda x: (x - mean)/std)
		
		if self.team_stats:
			pbp2 = pbp1.merge(tsaway,left_on = 'awayteam',right_on = 'TEAM',how = 'inner')
			pbp3 = pbp2.merge(tshome,left_on = 'hometeam',right_on = 'TEAM',how = 'inner')
		else:
			pbp3 = pbp1
		
		pbp3['game number'] = self.ct-1
		pbp3['game name'] = game_name
		if pbp3['awayscore'][len(pbp3)-1] > pbp3['homescore'][len(pbp3)-1]:
			isawaywin = 1
		else:
			isawaywin = 0
		pbp3['isawaywin'] = isawaywin
		return pbp3

	def __GetPlayersOnCourt(self,url):
		timeLength=2880

		gameName=url.split('/')[-1].split('.')[0]

		import requests
		from bs4 import BeautifulSoup
		import scipy as sp
		import scipy.interpolate

		page = requests.get(url).text
		soup = BeautifulSoup(page, 'lxml')
		table=soup.find_all("div",{"class":"plusminus"})

		players=[p.findAll("span")[0].string for p in table[0].find_all("div",{"class":'player'})]
		teams=[t.string for t in table[0].find_all("h3")]

		OT=0
		OTHeader=str(table[0].find_all("div",{"class":"header"})[0])

		if '4th OT' in OTHeader: 
			OT=4
		elif '3rd OT' in OTHeader: 
			OT=3
		elif '2nd OT' in OTHeader:
			OT=2
		elif '1st OT' in OTHeader:
			OT=1
		timeLengthTotal=timeLength+OT*300
		timeLength=int(timeLengthTotal/5)

		ct=-1
		players_info={}
		for player in table[0].findAll("div",{"class":"player-plusminus"}):
			ct+=1
			player_events=[]
			for event in player.findAll("div"):
				eventText=str(event)
				if 'plus' in eventText or 'minus' in eventText or 'even' in eventText:
					num=int(eventText.split("width:")[1].split('px')[0])
					player_events=player_events+[1]*num
				else:
					num=int(eventText.split("width:")[1].split('px')[0])
					player_events=player_events+[0]*num
			players_info[players[ct]]=player_events

		Team1BS=BeautifulSoup(str(table[0]).split('</h3>')[1], 'lxml')
		Team2BS=BeautifulSoup(str(table[0]).split('</h3>')[2], 'lxml')
		AwayPlayers=[p.findAll("span")[0].string for p in Team1BS.find_all("div",{"class":'player'})]
		HomePlayers=[p.findAll("span")[0].string for p in Team2BS.find_all("div",{"class":'player'})]

		Data=pd.DataFrame()
		for i in players_info:
			if i in HomePlayers:
				y=-1*np.array(players_info[i])
			else:
				y=players_info[i]
			x=np.linspace(0,1,len(y))
			new_x = np.linspace(0, 1, timeLength)
			new_y = np.around(sp.interpolate.interp1d(x, y, kind='linear')(new_x))
			Data[i]=new_y
		Data['T-'+teams[0]]=[1]*timeLength
		Data['T-'+teams[1]]=[-1]*timeLength
		Data['Time']=np.arange(timeLengthTotal,0,-5,dtype=int)
		Data['gameName']=gameName
		return Data

	def __WP_error(self,prob,y):
		WP_away = pd.DataFrame({'WP':prob[:,1], 'isawaywin':y})        
		sampling = 161;
		prob_array = np.linspace(0,1,sampling)
		prob_predict = []
		prob_true = []
		for k in range(len(prob_array)-1):
			prob_predict.append(np.mean(prob_array[k:k+1]))
			temp = WP_away['isawaywin'][WP_away['WP']<prob_array[k+1]][WP_away['WP']>prob_array[k]]
			prob_true.append(np.sum(temp)/len(temp))
		self.prob_predict = prob_predict
		self.prob_true = prob_true
		return (np.sum((np.array(prob_predict) - np.array(prob_true))**2)/(sampling-1))**0.5
	
	#### END INTERNAL FUNCTIONS ######

	def DownloadPlayerOnCourtInfo(self):

		ListOfFiles=pd.read_csv(self.DataFloder+'AllFilesList.csv')
		TotalLen=len(ListOfFiles['1'])
		ct=0
		AllData=[]
		if not os.path.isfile(self.DataFloder+'OnCourtPlayers.csv'):
			for i in ListOfFiles['1']:
				FileArr=i.split('pbp')
				url=FileArr[0]+'plus-minus'+FileArr[1]
				AllData.append(self.__GetPlayersOnCourt(url))
				ct+=1
				print('Plus Minus {}% Completed'.format(ct/TotalLen*100))
				
			pd.concat(AllData, ignore_index=True).fillna(0).to_csv(self.DataFloder+'OnCourtPlayers.csv')

		print('Dowloading On Court Player information Completed!')

	def ScrapeData(self):
		if not os.path.isfile(self.DataFloder+'AllFilesList.csv'):
			Seasons=self.AllSeasons
			AllMonths=['october','november','december','january','february','march','april']
			allLinks=[]
			for Season in Seasons:
				for Month in AllMonths:
					print('Scraping Month-{} Season-{}'.format(Month,Season))
					allLinks=allLinks+self.__GetLinks(Month,Season)
			self.__CreateNameCSV(allLinks)
		
		allFiles=pd.read_csv(self.DataFloder+'AllFilesList.csv').as_matrix()
		self.ct=0;total=len(allFiles)
		for File in allFiles:
			self.ct+=1 
			print('Creating Data CSV {}% Completed'.format(self.ct/total*100))
			if not os.path.isfile(self.DataFloder+File[1]):
				self.__CreateCSV(File[2])
		print('Scrape Data Completed!')

	def CreateAllFeaturesCSV(self):
		allLinks=pd.read_csv(self.DataFloder+'AllFilesList.csv')
		frames=[]
		self.ct=0;total=len(allLinks.as_matrix())
		if not os.path.isfile(self.DataFloder+'AllFeatures.csv'):
			for Link in allLinks.as_matrix():
				self.ct+=1 
				print('Creating Feature {}% Completed'.format(self.ct/total*100))
				frames.append(self.__CreatePBPFeatures(self.DataFloder+Link[1]))
			FeatureDF=pd.concat(frames,ignore_index=True)
			FeatureDF.to_csv(self.DataFloder+'AllFeatures.csv')
		print('Create Data CSV Completed!')

	def BuildDF(self):
		Data=pd.read_csv(self.DataFloder+'AllFeatures.csv')
		
		away_list = Data['awayteam']
		home_list = Data['hometeam']
		event = Data['event']
		self.game_time = Data['real time']
		self.game_name = Data['game name']
		
		DelArray=['Unnamed: 0','awayteam', 'hometeam', 'Defensive rebound',
			   'Offensive rebound', 'makes free throw', 'makes technical free throw',
			   'makes 2-pt shot', 'makes 3-pt shot', 'misses free throw',
			   'misses technical free throw', 'misses 2-pt shot', 'misses 3-pt shot',
			   'Personal foul', 'Personal take foul', 'Loose ball foul',
			   'Shooting foul', 'Technical foul', 'Turnover', 'full timeout',
			   'assist by', 'isawayevent','event','game name']
		for val in DelArray: del Data[val]
		
		if self.team_stats:
			del Data['TEAM_x']
			del Data['TEAM_y']

		Data['AdjustedScore1']= 10000*(Data['awayscore']-Data['homescore'])/(Data['time']+1)**1
		Data['AdjustedScore2']= 10000*(Data['awayscore']-Data['homescore'])/(Data['time']+1)**2
		Data['AdjustedScore3']= 10000*(Data['awayscore']-Data['homescore'])/(Data['time']+1)**3

		for name in ['free throw', '2-pt shot', '3-pt shot','technical free throw']:      
			Data['away '+name] = Data['count away makes '+name]+Data['count away misses '+name]
			Data['home '+name] = Data['count home makes '+name]+Data['count home misses '+name]
			Data['away '+name+' rate']= Data['count away makes '+name]/(Data['away '+name]+0.001)
			Data['home '+name+' rate']= Data['count home makes '+name]/(Data['home '+name]+0.001)
			del Data['count away makes '+name]
			del Data['count away misses '+name]
			del Data['count home makes '+name]
			del Data['count home misses '+name]
		del Data['time']
		conti_columns = ['count away Defensive rebound',
			   'count away Offensive rebound', 'away free throw',
			   'away technical free throw', 'away 2-pt shot',
			   'away 3-pt shot', 'count away Personal foul',
			   'count away Personal take foul', 'count away Loose ball foul',
			   'count away Shooting foul', 'count away Technical foul',
			   'count away Turnover', 'count away full timeout',
			   'count away assist by', 'count home Defensive rebound',
			   'count home Offensive rebound', 'home free throw',
			   'home technical free throw', 'home 2-pt shot',
			   'home 3-pt shot',  'count home Personal foul',
			   'count home Personal take foul', 'count home Loose ball foul',
			   'count home Shooting foul', 'count home Technical foul',
			   'count home Turnover', 'count home full timeout',
			   'count home assist by','awayscore','homescore']
		# standarize the continuous attributes
		for column in conti_columns:
			min_value=np.min(Data[column])
			max_value=np.max(Data[column])
			Data[column]=Data[column].map(lambda x: (x - min_value)/(max_value-min_value))
		self.DF=Data
		print('Build DF Completed!')
 
	def TrainTestSplit(self,train_fraction=0.5):
		df2=self.DF
		np.random.seed(9001)
		rand_array = np.random.rand(np.max(df2['game number'])+1)
		msk_train = rand_array < train_fraction
		game_number_list = np.arange(0,np.max(df2['game number'])+1)
		d = {'game number': game_number_list,
			'is_train': msk_train}
		train_frame = pd.DataFrame(d)
		df3 = df2.merge(train_frame,left_on = 'game number',right_on = 'game number',how = 'inner')

		# here we randomly split data based on game number
		data_train = df3[df3['is_train']]
		data_test = df3[~df3['is_train']]
		
		self.xTrain = data_train.drop(['isawaywin','is_train','game number','real time', 'quarter_real'],axis = 1).reset_index(drop=True)
		self.xTest = data_test.drop(['isawaywin','is_train','game number','real time', 'quarter_real'],axis = 1).reset_index(drop=True)
		
		self.yTrain = data_train['isawaywin'].reset_index(drop=True)
		self.yTest = data_test['isawaywin'].reset_index(drop=True)
		self.Gamenumber = df2['game number'] 
		print('Train-Test Split Completed!')
		
	def fitCV(self):
		n_folds = 2
		C_len = 20
		start = -4; end = 2;
		C_range = np.logspace(start,end,C_len)
		valid_acc = np.zeros((C_len,n_folds))

		# split test train sets multiple times 
		fold = 0
		for train_index, valid_index in KFold(n_folds, shuffle=True).split(range(len(self.xTrain))): # split data into train/test groups, 4 times
			Xtrain = self.xTrain.iloc[train_index]
			Xvalid = self.xTrain.iloc[valid_index]
			ytrain = self.xTest.iloc[train_index]
			yvalid = self.xTest.iloc[valid_index]
			for k in range(len(C_range)):
				# train
				C = C_range[k]
				clf = LogisticRegression(C=C)
				clf.fit(self.xTest,self.yTest)
				prob = clf.predict_proba(self.xTest)
				valid_acc[k,fold] = self.__WP_error(prob,self.yTest)       
			fold += 1
		# choose the p_th with minimal cost
		best_C = C_range[np.argmin(np.mean(valid_acc,axis=1))]
		print('CV Validation accuracy:',valid_acc)
		print('CV Best C:',best_C)
		print('CV Fit Analysis Completed!')
	
	def fit(self,C):
		clf = LogisticRegression(C=C)
		clf.fit(self.xTrain,self.yTrain)
		print('Train score: ', clf.score(self.xTrain,self.yTrain))
		error = self.__WP_error(clf.predict_proba(self.xTrain),self.yTrain)
		print('WP error: ', error)
		self.clf = clf
		print('Model Fit Completed!')
	
	def plot_WP_error(self):
		prob = self.clf.predict_proba(self.xTest)
		error = self.__WP_error(prob,self.yTest)
		prob_predict = self.prob_predict
		prob_true = self.prob_true
		plt.plot(prob_predict,prob_true, color='red',label = 'our prediction')
		plt.plot(prob_true,prob_true, color='blue',label = 'perfect prediction')   
		plt.ylabel('True probability')
		plt.xlabel('Predicted probability')
		plt.legend()
		print('WP mean squared error from perfect prediciton is: ', error*100, '%')
		print('Plot completed')
	
	def saveModelPersist(self,FileName):
		from sklearn.externals import joblib
		joblib.dump(self.clf, self.DataFloder+FileName) 
		print('Persist Model Completed!')
		
	def loadModelPersist(self,FileName):
		from sklearn.externals import joblib
		clf = joblib.load(self.DataFloder+FileName) 
		self.clf = clf
		
	def WP_list(self):
		prob = self.clf.predict_proba(self.xTest)
		WP_list = pd.DataFrame({'gameName':self.game_name.as_matrix(), 
								 'game time': self.game_time.as_matrix(), 
								 'WP':prob[:,1]})
		self.WP_list = WP_list
		
	def WP_game(self,game_name):
		plt.rcParams['figure.facecolor']='w'
		WP_list = self.WP_list
		WP_game = WP_list[WP_list.gameName == game_name]
		WP_game.plot(x='game time', y='WP',ylim=(0, 1))
		plt.ylabel('Away team win probability')
		plt.xlabel('Game time remaining')
		plt.axhline(y=0.5,ls='--',color = 'k',lw = 1)
		
	def player_ranking(self, threshold = 0,team_or_not = False):
		poc = pd.read_csv(self.DataFloder+'OnCourtPlayers.csv')
		WP_list = []; time_list = []; name_list = []
		WP_table = self.WP_list
		for gamename in WP_table['gameName'].unique():
			WP_sub = WP_table[WP_table.gameName == gamename]
			WP_q = list(WP_sub['WP'].as_matrix()); t_q = list(WP_sub['game time'].as_matrix());
			WP_q.append(WP_q[-1]); t_q.append(0); 
			f = interp1d(t_q, WP_q)
			t_seq = np.arange(t_q[0]-2.5, -0.25, -5)
			t_seq1 = np.arange(t_q[0], -5, -5)
			WP_seq = f(t_seq1)
			WP_seq = np.diff(WP_seq)
			name_seq = [gamename]*len(t_seq)
			WP_list = [*WP_list, *WP_seq]
			time_list = [*time_list, *t_seq]
			name_list = [*name_list, *name_seq]
		WP_table_new = pd.DataFrame({'gameName':name_list, 'game time': time_list, 'WP':WP_list})
		
		evaluator = pd.concat([poc, WP_table_new], axis=1)
		del evaluator['Unnamed: 0']
		del evaluator['gameName']
		del evaluator['game time']
		del evaluator['Time']
		
		# keep only the plays with large changes in WPs
		evaluator = evaluator[evaluator['WP'].abs()>threshold]
		
		X = evaluator.drop('WP',axis = 1)       
		y = evaluator['WP']
		X = X.fillna(0)
		X = sm.add_constant(X)
		if not team_or_not:
			team_name = []
			for name in evaluator.columns:
				if name[0:2] == 'T-':
					team_name.append(name)          
			X = X.drop(team_name,axis = 1)
		
		model = sm.OLS(y,X)
		results = model.fit()
		
		coef = pd.DataFrame(results.params,columns=['coef'])
		player_time = pd.DataFrame((X[X.columns].abs()).sum(),columns=['time'])

		play_evaluate = pd.concat([coef,player_time],axis=1)

		play_evaluate['total'] = play_evaluate['coef']* play_evaluate['time']

		play_evaluate['name'] = play_evaluate.index
		play_evaluate['Rank'] = play_evaluate['total'].rank(ascending = False)
		play_rank = play_evaluate.sort_values('Rank').reset_index(drop=True).dropna()
		# drop the const row
		play_rank.drop(play_rank.index[play_rank[play_rank['name'] == 'const'].index])
		return play_rank

class EDAImport:
	def __init__(self,FilePath):
		self.Data=pd.read_csv(FilePath)
		import seaborn as sns
		plt.rcParams['figure.facecolor']='w'

	def PrintKeys(self):
		print(self.Data.columns.get_values())

	def EDA(self,Key1Name,Key2Name,samplePoints):
		Data=self.Data.sample(samplePoints).copy()
		DataY=Data.isawaywin
		X1=Data[Key1Name]
		Y1=Data[Key2Name]

		color=[sns.color_palette()[0],sns.color_palette()[1],sns.color_palette()[2]]
		for k in range(len(DataY.as_matrix())):
			c1 = color[int(DataY.iloc[k])-1]
			plt.scatter(X1.iloc[k],Y1.iloc[k], c=c1, alpha=0.5)
		plt.xlabel(Key1Name)
		plt.ylabel(Key2Name)

	def EDA_YDiff(self,Key1Name,Key2Name,Key2NameDiff,samplePoints,yLabel):
		Data=self.Data.sample(samplePoints).copy()

		DataY=Data.isawaywin
		X1=Data[Key1Name]
		Y1=Data[Key2Name]
		Y2=Data[Key2NameDiff]
		YY=Y2-Y1

		color=[sns.color_palette()[0],sns.color_palette()[1],sns.color_palette()[2]]
		for k in range(len(DataY.as_matrix())):
			c1 = color[int(DataY.iloc[k])-1]
			plt.scatter(X1.iloc[k],YY.iloc[k], c=c1, alpha=0.5)
		plt.xlabel(Key1Name)
		plt.ylabel(yLabel)

	def EDA_XYDiff(self,Key1Name,Key1NameDiff,Key2Name,Key2NameDiff,samplePoints,xLabel,yLabel):
		Data=self.Data.sample(samplePoints).copy()

		DataY=Data.isawaywin
		X1=Data[Key1Name]
		X2=Data[Key1NameDiff]
		Y1=Data[Key2Name]
		Y2=Data[Key2NameDiff]
		YY=Y2-Y1
		XX=X2-X1

		color=[sns.color_palette()[0],sns.color_palette()[1],sns.color_palette()[2]]
		for k in range(len(DataY.as_matrix())):
			c1 = color[int(DataY.iloc[k])-1]
			plt.scatter(XX.iloc[k],YY.iloc[k], c=c1, alpha=0.5)
		plt.xlabel(xLabel)
		plt.ylabel(yLabel)
```

