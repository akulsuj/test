import logging
import shutil
import os
from Entities.Customentities import ApihomeResp
from Services import CustomException, dboperations as dbops
from datetime import datetime
import globalvars as gvar
import datetime
import re

dbops_obj = dbops.dboperations()
gvar.sadrd_settings = dbops_obj.SadrdSysSettings()
gvar.sadrd_ErrMessages = dbops_obj.SADRD_Sys_Message()

def ExtractFilesToLoad(folderpath, action):    
    inpLoadFolder = ""    
   
    # Get list of input files to process
    lsinpfiles = getinpfilenames_toprocess(folderpath,inpLoadFolder);
    return lsinpfiles

def getinpfilenames_toprocess(folderpath,inpLoadFolder):
    logging.debug("In getinpfilenames_toprocess() method")

    lsinpfiles = []  
    Inputdirpath = os.path.join(folderpath, inpLoadFolder)
    
    for filename in os.listdir(Inputdirpath):
         if filename.endswith(".xlsx") or filename.endswith(".xlsm") or filename.endswith(".csv"):
            lsinpfiles.append(os.path.join(Inputdirpath, filename))
            continue
         else:
            continue

    return lsinpfiles

def DownloadServerFilesToLoad(serverInputFilesByAction, Inputdirpath, action, year):
    logging.debug("In DownloadServerFilesToLoad() method")
    # Get list of input files to process
    lsinpfiles = Downloadfilenames_toprocess(serverInputFilesByAction, Inputdirpath, action, year);
    return lsinpfiles

def Downloadfilenames_toprocess(serverInputFilesByAction, Inputdirpath, action, inputYear):
    logging.debug("In Downloadfilenames_toprocess() method")
    try:
        lsinpfiles = []        
        inputFilesQtrList = []
        lstInpfilesVerified = []
        
        errFileImport = ""
        errFileImportWarning = ""

        response = ApihomeResp()
        
        if not os.path.exists(Inputdirpath):
            os.makedirs(Inputdirpath)

        companyList = []
        quartersList = []
        for (row, sadrdSysSetting) in enumerate(gvar.sadrd_settings):
            if sadrdSysSetting.settingName == "Valid_Company":
                companyList.append(sadrdSysSetting.settingValue)
            if sadrdSysSetting.settingName == "Valid_Quarter":
                quartersList.append(sadrdSysSetting.settingValue)

        if action =="Annual Stmt - Sch D":
            datatbImportTypes = [x for x in gvar.sadrd_settings if x.settingName == 'Filename_AnnStmtSchD']
            datatbImportType = datatbImportTypes[0].settingValue
        elif action =="QualPctFTC":
            datatbImportTypes = [x for x in gvar.sadrd_settings if x.settingName == 'Filename_QualFTC']
            datatbImportType = datatbImportTypes[0].settingValue
        elif action =="FTCGrossup":
            datatbImportTypes = [x for x in gvar.sadrd_settings if x.settingName == 'Filename_FTCGrossup']
            datatbImportType = datatbImportTypes[0].settingValue
      
        for filename in os.listdir(serverInputFilesByAction):
            if (filename.endswith(".csv") or filename.endswith(".xlsx") or filename.endswith(".xls") or filename.endswith(".xlsm")):
                if datatbImportType in filename and not filename.startswith("~$"):
                    lsinpfiles.append(filename) 

        for filename in lsinpfiles:
            errMessage =""            
            file_name_1 = filename.split(".")[0]
            if file_name_1 != None:
                #Next 4 lines extracts Company Name           
                removeSpacesFromFileName = "".join(re.split("[ ]*", file_name_1)) #Eg '2019JHUSASchDPart2'
                splitBySchD = removeSpacesFromFileName.split("SchD")#Eg ['2019JHUSA', 'Part2']
                get1stSplitValueAftersplitBySchD = splitBySchD[0]#Eg 2019JHUSA
                fileNameCompany = get1stSplitValueAftersplitBySchD[4:] #Eg JHUSA
            if action == "FTCGrossup":
                fileComponents = file_name_1.split("_")
                fileQuarter = fileComponents[1]
                inputFilesQtrList.append(fileQuarter)

            if (filename.endswith(".csv") or filename.endswith(".xlsx") or filename.endswith(".xls") or filename.endswith(".xlsm")):
                #if filename.startswith("~"):
                #f = open(serverInputFilesByAction +"\\" + filename,'r')
                #if(open(serverInputFilesByAction +"\\" + filename,'r').closed):
                if is_open(serverInputFilesByAction +"\\" + filename) == True:
                        errMessage = 'E006' + "," + filename + "," + "None"
                        errFileImport = errMessage if errFileImport == '' else (errFileImport + "; " + errMessage)

                if (action =="Annual Stmt - Sch D" and datatbImportType not in filename ) or (action =="QualPctFTC" and  datatbImportType not in filename) or (action =="FTCGrossup" and  datatbImportType not in filename):
                #if(action =="FTCGrossup" and  datatbImportType not in filename)
                    errMessage = 'E013' + "," + filename + "," + "None"
                    errFileImport = errMessage if errFileImport == '' else (errFileImport + "; " + errMessage)
                
                if (action =="Annual Stmt - Sch D" and datatbImportType in filename and not filename.endswith(".csv")):
                    errMessage = 'E003' + "," + filename + "," + "None"
                    errFileImport = errMessage if errFileImport == '' else (errFileImport + "; " + errMessage)
                
                if (action =="QualPctFTC" and datatbImportType in filename and (not filename.endswith(".xlsx") and not filename.endswith(".xls") and not filename.endswith(".xlsm"))):
                    errMessage = 'E003' + "," + filename + "," + "None"
                    errFileImport = errMessage if errFileImport == '' else (errFileImport + "; " + errMessage)

                if (action =="FTCGrossup" and datatbImportType in filename and (not filename.endswith(".xlsx") and not filename.endswith(".xls") and not filename.endswith(".xlsm"))):
                    errMessage = 'E003' + "," + filename + "," + "None"
                    errFileImport = errMessage if errFileImport == '' else (errFileImport + "; " + errMessage)
                
                if (action =="FTCGrossup" and datatbImportType in filename and (filename.endswith(".xlsx") or filename.endswith(".xls") or filename.endswith(".xlsm")) and fileQuarter not in quartersList):
                    errMessage = 'E013' + "," + filename + "," + "None"
                    errFileImport = errMessage if errFileImport == '' else (errFileImport + "; " + errMessage)
                
                if (action =="Annual Stmt - Sch D" and datatbImportType in filename and fileNameCompany not in companyList):
                    errMessage = 'E011' + "," + filename + "," + "None"
                    errFileImport = errMessage if errFileImport == '' else (errFileImport + "; " + errMessage) 
                                    
                if int(inputYear) != int(filename[0:4]):                    
                    errMessage = 'E012' + "," + filename + "," + "None"
                    errFileImport = errMessage if errFileImport == '' else (errFileImport + "; " + errMessage) 

                if errMessage == "":
                    shutil.copyfile(serverInputFilesByAction + '//' + filename, Inputdirpath + '//' + filename)
                    lstInpfilesVerified.append(os.path.join(Inputdirpath, filename))          
                continue 

        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, action, 'Downloadfilenames_toprocess', str(datetime.datetime.now())[0:23], "Downloadfilenames_toprocess method executed", None)
        
        if errFileImport != "":
            if len(lstInpfilesVerified) == 0:
                errNoFilesMessage = 'E002' + "," + "" + "," + ""
                errFileImport = errFileImport if errNoFilesMessage == '' else (errFileImport + "; " + errNoFilesMessage)
                lstInpfilesVerified = []
            errFileImport = errFileImport if errFileImportWarning == '' else (errFileImport + "; " + errFileImportWarning)
            errMessageComplete = dbops_obj.BuildErrorMessage(errFileImport)         
            raise CustomException.FileValidationException(errMessageComplete)
    except Exception as e: 
        dbops.logger.error('Error in Downloadfilenames_toprocess -' + str(e))
        logging.exception("Exception occurred in Downloadfilenames_toprocess() method :: " + str(e)) 
        raise e

    if len(lsinpfiles) == 0 or (action == "FTCGrossup" and len(lsinpfiles) != 4):
        if action == "FTCGrossup" and len(lsinpfiles) != 0:
            errMessage = 'E028' + "," + "" + "," + ""
        else:
            errMessage = 'E002' + "," + "" + "," + ""
        response.status = "Failure"
        response.message = dbops_obj.BuildErrorMessage(errMessage)
        dbops.logger.error(response.message)
        raise CustomException.FileValidationException(response.message)
    
    if (action == "FTCGrossup"):
        qtrValidationList = [item for item in quartersList if item not in inputFilesQtrList]   
        if len(qtrValidationList) != 0:
            errMessage = 'E032' + "," + ' | '.join(qtrValidationList) + "," + "None" 
            response.status = "Failure"
            response.message = dbops_obj.BuildErrorMessage(errMessage)
            dbops.logger.error(response.message)
            raise CustomException.FileValidationException(response.message)
    
    return lstInpfilesVerified

def is_open(file_name):
    if os.path.exists(file_name):
        try:
            os.rename(file_name, file_name) #can't rename an open file so an error will be thrown
            return False
        except:
            return True
    raise NameError
