# AutomatizedEEGLABScript

%% Automatized analysis using EEGLAB
% written by Xiaoping Fang (xpfang4@gmail.com)
% Data preparation: markup events in NetStation; then expoert data with
% simple binary formt
% rename data with Subject ID (e.g., 1001) using EP toollkit (from
% continuous .raw data to eeglab format (advantage no need to reload channel location
% file as EP toolkit can figure the correct location file)
% the script does not save every file after every step, instead these files
% are saved: 
% (1) raw data file with correct channle location file (subject ID); 
% (2) downsampled (to 500HZ), filterred and line noise clearned file (subject ID_PreRejection
% (3) after manual rejection of continuous data (Subject ID_Rejected);
% (4) after bad channel detection, ASR correction, and bad channel
% intepolation and reference (Subject_PreICA);
% (5) after ICA (SubjectID_PostICA);
%% Basic set up and define parameters
if ismac
   StudyPath = '/Volumes/Samsung_T3/WordLearning_Fall2017/Xiaoping/EEGLAB/';
elseif ispc
   StudyPath = 'F:\WordLearning_Fall2017\Xiaoping\EEGLAB\';
elseif isunix
    StudyPath = 'tobedefined'; % not necessary if you never switch to Linux
else
    disp('Platform not supported')
end

ChannelLocationFile = '/Users/xif20/Documents/MATLAB/eeglab/sample_locs/GSN-Hydrocel-129.ced';

StudyName = 'WordLearning_Exp7_ERP.study';
StudyNameShort = 'Exp7';
TaskName = 'SemanticJudgment';
StudyNote = 'Embodied';

AllSubject = {...
    '1001', '1003' '1004' '1005' '1006' '1007' '1008' '1009' '1010' '1011' '1013' '1014'  ...
    '1016' '1017' '1018' '1019' '1020' '1021' '1022' '1023' '1024' '1025' ...
    '1026' '1027' '1030' '1031' ...
    }; % values are used as subject ID 
Condition = {...
    'NovelAction', 'NovelVisual', ...
    'RealAction', 'RealVisual',...
    }; % 4 conditions: 2 by 2

BandPass = [0.1 80]; % for filter, cut off frequency: 0.1 t0 80.5 HZ

cd(StudyPath) % go to the folder with all the data files
STUDY = []; CURRENTSTUDY = 0; ALLEEG = []; EEG=[]; CURRENTSET=[]; % initilize
 [ALLEEG EEG CURRENTSET ALLCOM] = eeglab; % start EEGLAB under Matlab 
%% load the correct channel location file and save the raw data
% data import and load the correct channel data file
% detect bad channels: 5SD and take notes on bad channels
for i = 6:length(AllSubject)
    RawDataFile = strcat(StudyPath,AllSubject{i},'.raw');
    [ALLEEG EEG CURRENTSET ALLCOM] = eeglab;
    EEG = pop_readegi(RawDataFile, [],[],'auto');
    [ALLEEG EEG CURRENTSET] = pop_newset(ALLEEG, EEG, 0,'setname',AllSubject{i},'gui','off'); 
    EEG=pop_chanedit(EEG, 'load',{ChannelLocationFile 'filetype' 'autodetect'},'changefield',{129 'datachan' 0});
    EEG.comments = pop_comments(EEG.comments,'','Dataset has the correct channel location file.',1);
    [ALLEEG EEG CURRENTSET ] = eeg_store(ALLEEG, EEG);% Store the dataset into EEGLAB
    EEG = pop_saveset( EEG, 'filename',strcat(AllSubject{i},'.set'),'filepath',StudyPath);
    ALLEEG = []; EEG=[]; CURRENTSET=[];
end

%% resample and filter the data, and clean line noise to get ready for manual rejection
% downsample data to 250HZ, to speed up (ok if highest frequency you care
% about is no higher than 250/3 HZ
% band-pass at 0.2 to 80 HZ, to reduce low frequency drift 

for i = 1:length(AllSubject)
    EEG = pop_loadset('filename',strcat(AllSubject{i},'.set'),'filepath',StudyPath);
    EEG = pop_resample( EEG, 250); % data are resampled to 500 HZ
    EEG.comments = pop_comments(EEG.comments,'','Dataset was downsampled to 250HZ.',1); 
    EEG.setname = AllSubject{i};
    EEG = pop_eegfiltnew(EEG, BandPass(1),BandPass(2),[],0,[],0); % filter order is automaitcally calculated based on sampling rate and the target frequency 
    EEG.comments = pop_comments(EEG.comments,'',strcat('Data were filtered at ',BandPass(1),'_',BandPass(2), ' HZ'),1);
    [ALLEEG EEG CURRENTSET ] = eeg_store(ALLEEG, EEG);% Store the dataset into EEGLAB
    EEG = pop_cleanline(EEG, 'bandwidth',2,'chanlist',[1:128] ,'computepower',1,'linefreqs',[60 120] ,'normSpectrum',0,'p',0.01,'pad',2,'plotfigures',0,'scanforlines',1,'sigtype','Channels','tau',100,'verb',1,'winsize',4,'winstep',1);
    EEG.comments = pop_comments(EEG.comments,'','Line noise was cleaned.',1);
    [ALLEEG EEG CURRENTSET ] = eeg_store(ALLEEG, EEG);% Store the dataset into EEGLAB
    EEG = pop_saveset( EEG, 'filename',strcat(AllSubject{i},'_PreRejection.set'),'filepath',StudyPath);
    ALLEEG = []; EEG=[]; CURRENTSET=[];
end

%% manual rejction, detect bad channels, using ASR to clean continuous data, intepolate bad channels, and reference to average
% manual rejection of bad portions of continuous data: take out unusual
% data while keep stereotical artifacts

for i = 4:length(AllSubject)
    EEG = pop_loadset('filename',strcat(AllSubject{i},'_PreRejection.set'),'filepath',StudyPath)
    eeglab redraw;
    pop_eegplot( EEG, 1, 1, 1);
    ud=get(gcf,'UserData');
    ud.winlength=60;
    set(gcf,'UserData',ud);     
    eegplot('draws',0)          

    uiwait; % stop for manual rejection of continuous data AND check bad electrodes
    
    EEG.comments = pop_comments(EEG.comments,'','Manual rejection was done on continuous data.',1);
    EEG = pop_saveset( EEG, 'filename',strcat(AllSubject{i},'_Rejected.set'),'filepath',StudyPath);
    ALLEEG = []; EEG=[]; CURRENTSET=[];
end

%% data clean: detect bad channels, using ASR to correct extreme values; replace bad channels; re-reference to average and run ICA
ALLEEG = []; EEG=[]; CURRENTSET=[];

for i = 7:length(AllSubject)
    EEG = pop_loadset('filename',strcat(AllSubject{i},'_Rejected.set'),'filepath',StudyPath);
    [ALLEEG EEG CURRENTSET] = eeg_store(ALLEEG, EEG);
    EEG = clean_rawdata(EEG, -1, [-1], 0.7, -1, 20, 0.25); % use SD of 20 to avoid over clean the data
    EEG.comments = pop_comments(EEG.comments,'','Data were cleaned with ASR.',1);
    [ALLEEG EEG CURRENTSET] = eeg_store(ALLEEG, EEG);
    % replace bad channels if necessary
    if EEG.nbchan < 128     
    ChanToReplace = setdiff({EEG.urchanlocs.labels},{EEG.chanlocs.labels});
    ChanNumToReplace = zeros(1,length(ChanToReplace));
    for k = 1 : length(ChanNumToReplace)
        cellContents = ChanToReplace{k};
        % Truncate and stick back into the cell
        ChanNumToReplace(k) = str2num(cellContents(2:end));
    end
    validchan = 128 - length(ChanNumToReplace);
    EEG = pop_interp(EEG, ALLEEG(1).chanlocs(ChanNumToReplace), 'spherical'); % get the values from channels defined in the previous step (may need to use other chanlocas file info.
    EEG.comments = pop_comments(EEG.comments,'',strcat('Bad channels were deleted and intepolated: ',num2str(ChanNumToReplace)),1);
    else
        validchan = 128;
    end
    EEG = pop_reref( EEG, [],'refloc',struct('labels',{'E129'},'theta',{0},'radius',{0},'X',{5.4497e-16},'Y',{0},'Z',{8.9},'sph_theta',{0},'sph_phi',{90},'sph_radius',{8.9},'type',{'REF'},'ref',{''},'urchan',{129},'datachan',{0}));
    EEG.comments = pop_comments(EEG.comments,'','Data were rereferenced to average.',1);
    EEG = pop_saveset( EEG, 'filename',strcat(AllSubject{i},'_PreICA.set'),'filepath',StudyPath);

    % run ICA, no pca is applied for now
    eeglab redraw;
    EEG = pop_runica(EEG, 'extended',1,'interupt','on');
    [ALLEEG EEG] = eeg_store(ALLEEG, EEG, CURRENTSET);
    EEG = pop_saveset( EEG, 'filename',strcat(AllSubject{i},'_PostICA.set'),'filepath',StudyPath);
    ALLEEG = []; EEG=[]; CURRENTSET=[];
end


%% New procedure: run ICA, remove components, then interpolate bad channels and reference the data

ALLEEG = []; EEG=[]; CURRENTSET=[];


for i = 3:length(AllSubject)
    EEG = pop_loadset('filename',strcat(AllSubject{i},'_Rejected.set'),'filepath',StudyPath);
    [ALLEEG EEG CURRENTSET] = eeg_store(ALLEEG, EEG);
    EEG = clean_rawdata(EEG, -1, [-1], 0.7, -1, 20, 0.25); % use SD of 20 to avoid over clean the data
    EEG.comments = pop_comments(EEG.comments,'','Data were cleaned with ASR.',1);
    [ALLEEG EEG CURRENTSET] = eeg_store(ALLEEG, EEG);
    EEG = pop_saveset( EEG, 'filename',strcat(AllSubject{i},'_ASR.set'),'filepath',StudyPath);
    EEG = pop_runica(EEG, 'extended',1,'interupt','on');
    EEG = pop_saveset( EEG, 'filename',strcat(AllSubject{i},'_ICA_First.set'),'filepath',StudyPath);
    ALLEEG = []; EEG=[]; CURRENTSET=[];
end

%% Use Adjust to faciliate the detection of artifact ICs.

    EEG = pop_loadset('filename',strcat(AllSubject{i},'_ICA_First.set'),'filepath',StudyPath);
    eeglab redraw;
    pop_ADJUST_interface(strcat(AllSubject{i},'_ICA_First.set'));
     

%% after removing artifact components, intepolate the channels and reference the data



%% create a study with no design to make compoment rejection easier?
% to check ICA results: double check the parameters are good for every
% participants

    EEG = pop_loadset('filename',strcat(AllSubject{i},'_PostICA.set'),'filepath',StudyPath);
    [ALLEEG EEG CURRENTSET] = eeg_store(ALLEEG, EEG);

% use Adjust for suggestions about ICs to remove
%%
EEG = eeg_checkset( EEG );
pop_eegplot( EEG, 0, 1, 1);
EEG = eeg_checkset( EEG );
pop_eegplot( EEG, 1, 1, 1);
