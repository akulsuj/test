import logging
import os
import shutil
from flask import Flask, request, jsonify
from flask_cors import CORS
from datetime import datetime

import globalvars as gvar
import json
import datetime
from pathlib import Path
import urllib
from db import db
import Services.dboperations as dbops
import Services.Auth as auth
from Entities.Customentities import ApihomeResp
import pandas as pd
from Services.logoperations import insertServerEventLog

mainapp = Flask(__name__)

if mainapp.config["ENV"] == "production":
    mainapp.config.from_object("config.ProductionConfig")
elif mainapp.config["ENV"] == "stage":
    mainapp.config.from_object("config.StageConfig")
elif mainapp.config["ENV"] == "test":
    mainapp.config.from_object("config.TestingConfig")
elif mainapp.config["ENV"] == "development":    
    mainapp.config.from_object("config.DevelopmentConfig")
else:
    mainapp.config.from_object("config.LocalConfig")

cors = CORS(mainapp, resources={r"/*": {"origins": json.loads(mainapp.config["CORS_ORIGINS"])[mainapp.config["ENV"]]}}, supports_credentials=True)

gvar.init()
gvar.gconfig = mainapp.config
gvar.sqlconfig = urllib.parse.quote_plus("DRIVER="+gvar.gconfig["DRIVER"] + ";" + "SERVER=" + gvar.gconfig["SADRD_DATABASE_SERVER"] + ";" + "DATABASE=" + gvar.gconfig["SADRD_DATABASE_NAME"] + gvar.gconfig["CONNECTION_AUTH_STRING"])

os.makedirs(str(Path.cwd().parent.joinpath('Logs')), exist_ok=True)
logging.basicConfig(level=mainapp.config["LOGLEVEL"],
                    format='%(asctime)s %(name)-12s %(levelname)-8s %(message)s',
                    datefmt='%m-%d %H:%M',
                    filename=str(Path.cwd().parent.joinpath('Logs','sadrd.log')),
                    filemode='a')

console = logging.StreamHandler()
console.setLevel(logging.INFO)
# Set a format which is simpler for console use
formatter = logging.Formatter('%(name)-12s: %(levelname)-8s %(message)s')
console.setFormatter(formatter)
# Add the handler to the root logger
logging.getLogger('').addHandler(console)

dbops_obj = dbops.dboperations()
gvar.sadrdUsersList = dbops_obj.GetAllUsers()
gvar.sadrd_ErrMessages = dbops_obj.SADRD_Sys_Message()

mainapp.config['SQLALCHEMY_DATABASE_URI'] = gvar.gconfig["SQLALCHEMYODBC"] % gvar.sqlconfig
mainapp.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
mainapp.config['PROPAGATE_EXCEPTIONS'] = True

with mainapp.app_context():
    db.init_app(mainapp)

logging.debug(gvar.sqlconfig)

@mainapp.after_request
def AddHeader(response):
    response.headers["Cache-Control"] = "no-store"
    response.headers["Pragma"] = "no-cache"
    response.headers["X-Content-Type-Options"] = "nosniff"
    response.headers["Content-Security-Policy"] = "script-src 'strict-dynamic' 'nonce-SadrdApiNonce' 'unsafe-inline' https:; object-src 'none'; base-uri 'none'; require-trusted-types-for 'script';"
    response.headers["Referrer-Policy"] = "no-referrer"
    response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
    response.headers["X-Frame-Options"] = "DENY"
    return response

@mainapp.route('/api/AuthenticateUser', methods=['GET'])
@auth.token_required
def AuthenticateUser():

    logging.debug("In AuthenticateUser() method")

    try:
        data = request.headers['Authorization']
        userNetworkId = auth.GetLoggedInUser(data)
        logging.debug("User Network Id :: " + userNetworkId)

        gvar.sadrdUsersList = dbops_obj.GetAllUsers()
        gvar.sadrd_settings = dbops_obj.SadrdSysSettings() 

        if not data or len(data) < 100 or str(data)[0:7] != 'Bearer ':
            dbops.logger.error ("In AuthenticateUser: Auth header missing or contains no bearer token")
            dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - AuthenticateUser()',
                        'AuthenticateUser', str(datetime.datetime.now())[0:23], ("Unable to check user authorization: no auth token supplied" + datatb[0].Name), None)
            loggedInUserDetail = dict([('authenticated', False)]) 
            logging.debug("In AuthenticateUser: Auth header missing or contains no bearer token")
            insertServerEventLog(mainapp.env, mainapp.config['SERVER_LOGFILES_FOLDER'], "Unable to check user authorization: Auth header missing or contains no bearer token", "Warning", userNetworkId,
                                    mainapp.config['API_ENDPOINT'], gvar.func, "User Authorization", gvar.user_ip_address, mainapp.config['APP_SERVER_IP_ADDRESS'], "STA-025-07")
        else:
            if len(gvar.sadrdUsersList):
                roleType = ''
                rolesList = GetAllRoles().json.get('RolesData')
                datatb = [x for x in gvar.sadrdUsersList if x.NetworkId == userNetworkId]
                roleType = [x for x in rolesList if x['RoleID'] == datatb[0].RoleId][0]['Type']
                if len(datatb):    
                    loggedInUserDetail = dict([('authenticated', True), ('NetworkId', datatb[0].NetworkId), ('Name', datatb[0].Name),
                                            ('RoleId', datatb[0].RoleId), ('IsActive', datatb[0].isActive), ('Email', datatb[0].Email), ('Role', roleType)])
                    logging.debug("In AuthenticateUser: User is Authorized - " + datatb[0].Name)
                    insertServerEventLog(mainapp.env, mainapp.config['SERVER_LOGFILES_FOLDER'], "User is Authorized", "Information", userNetworkId,
                                            mainapp.config['API_ENDPOINT'], gvar.func, "User Authorization", gvar.user_ip_address, mainapp.config['APP_SERVER_IP_ADDRESS'], "STA-025-07")
                else:
                    loggedInUserDetail = dict([('authenticated', False)]) 
                    logging.debug("In AuthenticateUser: User not Authorized")
                    insertServerEventLog(mainapp.env, mainapp.config['SERVER_LOGFILES_FOLDER'], "User not Authorized", "Warning", userNetworkId,
                                            mainapp.config['API_ENDPOINT'], gvar.func, "User Authorization", gvar.user_ip_address, mainapp.config['APP_SERVER_IP_ADDRESS'], "STA-025-07")
            else:
                loggedInUserDetail = dict([('authenticated', False)]) 
                logging.debug("In AuthenticateUser: User not Authorized")
                dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - AuthenticateUser()', 'AuthenticateUser', str(datetime.datetime.now())[0:23], ("User not Authorized" + datatb[0].Name), None)
                insertServerEventLog(mainapp.env, mainapp.config['SERVER_LOGFILES_FOLDER'], "User not Authorized", "Warning", userNetworkId,
                                            mainapp.config['API_ENDPOINT'], gvar.func, "User Authorization", gvar.user_ip_address, mainapp.config['APP_SERVER_IP_ADDRESS'], "STA-025-07")
    except Exception as e:
        logging.exception('Exception occurred in AuthenticateUser() :: ' + str(e))
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - AuthenticateUser()', 'AuthenticateUser', str(datetime.datetime.now())[0:23], ("Authentication failed :: " + str(e)), None)
        dbops.logger.error('Exception occurred in AuthenticateUser' + str(e))      

    return loggedInUserDetail

@mainapp.route('/api/GetImportTypes', methods=['GET'])
@auth.token_required
def GetImportTypes():

    logging.debug("In GetImportTypes() method")

    importTypesList = []

    try:
        importId = 0
        gvar.sadrd_settings = dbops_obj.SadrdSysSettings() 
        for (row, sadrdSysSetting) in enumerate(gvar.sadrd_settings):
            if sadrdSysSetting.settingName == "ImportType":
                importId += 1
                dictImportTypes = dict([('ImportId', importId), ('ImportType', sadrdSysSetting.settingValue), ('Description', sadrdSysSetting.Description)]) 
                importTypesList.append(dictImportTypes)             
    except Exception as e:
        logging.exception('Exception occurred in GetImportTypes() :: ' + str(e))
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - GetImportTypes()', 'GetImportTypes', str(datetime.datetime.now())[0:23], ("Exception occurred in GetImportTypes() :: " + str(e)), None)

    return jsonify( {'ImportTypesData' : importTypesList} )

@mainapp.route('/api/ImportData', methods=['POST'])
@auth.token_required
def ImportData():
    errInputFilesMessage = ""
    logging.debug("In ImportData() method") 

    from Services.parentparser import parentparser 
    
    response = ApihomeResp()
    reqForm = dict(request.form)

    year =  reqForm['year']
    importType =  reqForm['importType']

    gvar.sadrd_settings = dbops_obj.SadrdSysSettings()
    serverFolderPath = [x for x in gvar.sadrd_settings if x.settingName == "ServerFolderPath"][0].settingValue

    Inputdirpath = Path.cwd().joinpath('Data', importType)
    serverInputFilesByAction = Path.cwd().joinpath(serverFolderPath, "")
        
    try:
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, importType, 'APIHome() - ImportData()' , str(datetime.datetime.now())[0:23], "ImportData method called", None,0)
        gvar.sqlconfig = urllib.parse.quote_plus("DRIVER="+gvar.gconfig["DRIVER"] + ";" + "SERVER=" + gvar.gconfig["SADRD_DATABASE_SERVER"] + ";" + "DATABASE=" + gvar.gconfig["SADRD_DATABASE_NAME"] + gvar.gconfig["CONNECTION_AUTH_STRING"])
        response = parentparser(str(serverInputFilesByAction), str(Inputdirpath), importType, year, 'Yes')
        if (response.status == "Success"):
            dbops_obj.UpdateReportStatus('Y','SADRD_FileImported', "Import")
            dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - ImportData()', importType, str(datetime.datetime.now())[0:23], response.message , None,1)
            insertServerEventLog(mainapp.env, mainapp.config['SERVER_LOGFILES_FOLDER'], response.message, "Information", gvar.user_id, mainapp.config['API_ENDPOINT'],
                                    gvar.func, "Import Files", gvar.user_ip_address, mainapp.config['APP_SERVER_IP_ADDRESS'], "STA-025-01") 
        if (response.status == "Failure"):
            dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - ImportData()', importType, str(datetime.datetime.now())[0:23], response.message , None,1)
            insertServerEventLog(mainapp.env, mainapp.config['SERVER_LOGFILES_FOLDER'], response.message, "Warning", gvar.user_id, mainapp.config['API_ENDPOINT'],
                                    gvar.func, "Import Files", gvar.user_ip_address, mainapp.config['APP_SERVER_IP_ADDRESS'], "STA-025-01") 
        errInputFilesMessage = response.message
    except Exception as e:
        response.status = "Failure"
        errMessage = 'E031' + "," + "" + "," + ""
        errInputFilesMessage = dbops_obj.BuildErrorMessage(errMessage)
        logging.exception('Exception occurred in ImportData() :: ' + str(e))
        dbops.logger.error('Exception occurred in ImportData() :: ' + importType + ' :: ' + str(e))
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - ImportData()' , importType, str(datetime.datetime.now())[0:23], (errInputFilesMessage), None, 1)
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - ImportData()' , importType, str(datetime.datetime.now())[0:23], (str(e)), None, 0)
        insertServerEventLog(mainapp.env, mainapp.config['SERVER_LOGFILES_FOLDER'], ('Exception occurred in ImportData() :: ' + errInputFilesMessage), "Warning", gvar.user_id, mainapp.config['API_ENDPOINT'],
                                gvar.func, "Import Files", gvar.user_ip_address, mainapp.config['APP_SERVER_IP_ADDRESS'], "STA-025-01") 

    return jsonify({'status': response.status, 'message': errInputFilesMessage})

@mainapp.route('/api/GetActionLogs', methods=['GET'])
@auth.token_required
def GetActionLogs():

    logging.debug("In GetActionLogs() method") 

    dictActionLogList = []

    try:
        actionLogsList = dbops_obj.get_actionLog()
        for (row, actionLog) in enumerate(actionLogsList):
            dictActionLogs = dict([('LogID', actionLog.LogID), ('Month', actionLog.Month), ('Year', actionLog.Year), ('UserID', actionLog.UserID), ('Module', actionLog.Module), ('Action', actionLog.Action), ('ActionDate', actionLog.ActionDate), ('Comments', actionLog.Comments), ('Dataload_Id', actionLog.Dataload_Id),])    
            dictActionLogList.append(dictActionLogs)
        dictActionLogList.sort(key=lambda x:x['LogID'], reverse=True)
    except Exception as e:
        logging.exception('Exception occurred in GetActionLogs() :: ' + str(e))
        
    return jsonify({'ActionLogsData' : dictActionLogList})

@mainapp.route('/api/GetAllUserDetails', methods=['GET'])
@auth.token_required
def GetAllUserDetails():

    logging.debug("In GetAllUserDetails() method") 

    dictUsersList = []
    userDetailsList = []

    try:
        usersList = dbops_obj.GetAllUsers()
        dictRolesList = GetAllRoles().json.get('RolesData')
        for (row, user) in enumerate(usersList):
            dictUsers = dict([('NetworkId', user.NetworkId), ('Name', user.Name), ('isActive', user.isActive), ('RoleId', user.RoleId), ('Email', user.Email)])
            dictUsersList.append(dictUsers)
        df1 = pd.DataFrame(dictUsersList)
        df2 = pd.DataFrame(dictRolesList)
        userDetailsList = (df1.join(df2.set_index('RoleID'), on='RoleId')).to_dict('records')
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - GetAllUserDetails()', 'GetAllUserDetails', str(datetime.datetime.now())[0:23],'GetAllUserDetails method executed', None, 0)        
    except Exception as e:
        logging.exception('Exception occurred in GetAllUserDetails() :: ' + str(e))
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - GetAllUserDetails()', 'GetAllUserDetails', str(datetime.datetime.now())[0:23], ("Exception occurred in GetAllUserDetails() :: " + str(e)), None)

    return jsonify({'UsersData' : userDetailsList})

@mainapp.route('/api/GetAllRoles', methods=['GET'])
@auth.token_required
def GetAllRoles():

    logging.debug("In GetAllRoles() method") 

    dictRolesList = []
    
    try:
        rolesList = dbops_obj.GetAllRoles()
        for (row, role) in enumerate(rolesList):
            dictRoles = dict([('RoleID', role.RoleID), ('Type', role.Type)])    
            dictRolesList.append(dictRoles)
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - GetAllRoles()', 'GetAllRoles', str(datetime.datetime.now())[0:23],'GetAllRoles method executed', None, 0)        
    except Exception as e:
        logging.exception('Exception occurred in GetAllRoles() :: ' + str(e))
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - GetAllRoles()', 'GetAllRoles', str(datetime.datetime.now())[0:23], ("Exception occurred in GetAllRoles() :: " + str(e)), None)

    return jsonify({'RolesData' : dictRolesList})

@mainapp.route('/api/UpdateUser', methods=['POST'])
@auth.token_required
def UpdateUser():
    errSysMessage = ""
    logging.debug("In UpdateUser() method") 

    userAddOrUpdate = ''
    response = ApihomeResp()

    try:        
        response.status = dbops_obj.UpdateUser(request.form)
        if response.status == "Exists":
            userAddOrUpdate = 'Update User'
            databaseMsg = [x for x in gvar.sadrd_ErrMessages if x.MessageNumber == 'E023'][0]
            keyValueMsg = request.form['Name']
            response.message = databaseMsg.Message.replace('[insert key-values here]', keyValueMsg) + " " + databaseMsg.Action
        else:
            databaseMsg = [x for x in gvar.sadrd_ErrMessages if x.MessageNumber == 'E021'][0]
            if ("add" == request.form['userAction']):
                userAddOrUpdate = 'Add User'
                keyValueMsg = request.form['Name']
                response.message = databaseMsg.Message.replace('[Record]', keyValueMsg).replace('[UserAction]', 'added') + " " + databaseMsg.Action
            else:
                userAddOrUpdate = 'Update User'
                keyValueMsg = request.form['Name']
                response.message = databaseMsg.Message.replace('[Record]', keyValueMsg).replace('[UserAction]', 'updated') + " " + databaseMsg.Action
        errSysMessage = response.message
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - UpdateUser()', userAddOrUpdate, str(datetime.datetime.now())[0:23], response.message, None,1)
        insertServerEventLog(mainapp.env, mainapp.config['SERVER_LOGFILES_FOLDER'], response.message, "Information", gvar.user_id, mainapp.config['API_ENDPOINT'],
                                gvar.func, userAddOrUpdate, gvar.user_ip_address, mainapp.config['APP_SERVER_IP_ADDRESS'], "STA-025-06")
    except Exception as e:
        response.status = "Failure"
        response.message = "Exception occurred while updating the user - " + request.json['Name']
        logging.exception('Exception occurred in UpdateUser() :: ' + str(e))
        dbops.logger.error('Exception occurred while updating the user :: ' + str(e))
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - UpdateUser()', 'User Add/Update', str(datetime.datetime.now())[0:23], ("Exception occurred while Updating User - " + str(e)), None, 0)
        errMessage = 'E031' + "," + "" + "," + ""
        errSysMessage = dbops_obj.BuildErrorMessage(errMessage)
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - UpdateUser()', 'User Add/Update', str(datetime.datetime.now())[0:23], ("Exception occurred while Updating User - " + errSysMessage), None, 1)
        insertServerEventLog(mainapp.env, mainapp.config['SERVER_LOGFILES_FOLDER'], response.message, "Warning", gvar.user_id, mainapp.config['API_ENDPOINT'],
                                gvar.func, userAddOrUpdate, gvar.user_ip_address, mainapp.config['APP_SERVER_IP_ADDRESS'], "STA-025-06")
        
    return jsonify({'status': response.status, 'message': errSysMessage})

@mainapp.route('/api/CusipMappingData', methods=['GET'])
@auth.token_required
def CusipMappingData():
    errSysMessage = ""
    logging.debug("In CusipMappingData() method") 

    from Services.parentparser import parentparser

    gvar.sadrd_settings = dbops_obj.SadrdSysSettings()
    year = [x for x in gvar.sadrd_settings if x.settingName == "SADRD_Year"][0].settingValue

    response = ApihomeResp()
    importType = "generateSADRDReport"
    reportStatus = [x for x in gvar.sadrd_settings if x.settingName == "SADRD_FileImported"][0].settingValue

    if (reportStatus == 'N'):
        response.status = "Failure"
        databaseMsg = [x for x in gvar.sadrd_ErrMessages if x.MessageNumber == 'E007'][0]
        response.message = databaseMsg.Message + " " + databaseMsg.Action
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - CusipMappingData()', 'Process SADRD', str(datetime.datetime.now())[0:23],response.message, None, 1)
        errSysMessage = response.message
    else:
        try:
            Inputdirpath = Path.cwd().joinpath('Data', importType)
            serverInputFilesFolder = json.loads(mainapp.config["SADRD_SERVER_FOLDER"])
            serverInputFilesByAction = Path.cwd().joinpath(str(serverInputFilesFolder['serverFolderPath']), importType)
            response = parentparser(str(serverInputFilesByAction), str(Inputdirpath), importType, year, 'No')
            if (response.status == "Success"):
                dbops_obj.UpdateReportStatus('Y','SADRD_ReportGenerated', "Process SADRD")
                dbops_obj.UpdateReportStatus(str(datetime.datetime.now())[0:16], 'LastProcessSADRDRun', "Process SADRD")
            databaseMsg = [x for x in gvar.sadrd_ErrMessages if x.MessageNumber == 'E014'][0]
            dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'Process SADRD', 'Process SADRD', str(datetime.datetime.now())[0:23],databaseMsg.Message, None, 1)
            insertServerEventLog(mainapp.env, mainapp.config['SERVER_LOGFILES_FOLDER'], response.message, "Information", gvar.user_id, mainapp.config['API_ENDPOINT'],
                                    gvar.func, "Process SA DRD", gvar.user_ip_address, mainapp.config['APP_SERVER_IP_ADDRESS'], "STA-025-01")
            errSysMessage = response.message
        except Exception as e:
            response.status = "Failure"
            logging.exception('Exception occurred in CusipMappingData() :: ' + str(e))
            dbops.logger.error('Exception occurred in CusipMappingData()' + str(e))
            gvar.sqlconfig = urllib.parse.quote_plus("DRIVER="+gvar.gconfig["DRIVER"] + ";" + "SERVER=" + gvar.gconfig["SADRD_DATABASE_SERVER"] + ";" + "DATABASE=" + gvar.gconfig["SADRD_DATABASE_NAME"] + gvar.gconfig["CONNECTION_AUTH_STRING"])
            dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'Process SADRD', 'Process SADRD', str(datetime.datetime.now())[0:23], ("Exception occurred while Processing SADRD - " + str(e)), None, 0)
            errMessage = 'E031' + "," + "" + "," + ""
            errSysMessage = dbops_obj.BuildErrorMessage(errMessage) 
            dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'Process SADRD', 'Process SADRD', str(datetime.datetime.now())[0:23], ("Exception occurred while Processing SADRD - " + errSysMessage), None, 1)
            insertServerEventLog(mainapp.env, mainapp.config['SERVER_LOGFILES_FOLDER'], errSysMessage, "Warning", gvar.user_id, mainapp.config['API_ENDPOINT'],
                                    gvar.func, "Process SA DRD", gvar.user_ip_address, mainapp.config['APP_SERVER_IP_ADDRESS'], "STA-025-01")
    
    return jsonify({'status': response.status, 'message': errSysMessage})

@mainapp.route('/api/GetTaxYear', methods=['GET'])
@auth.token_required
def ReturnTaxYear():

    logging.debug("In ReturnTaxYear() method")

    try:
        gvar.sadrd_settings = dbops_obj.SadrdSysSettings()
        dataSettingYear = [x for x in gvar.sadrd_settings if x.settingName == 'SADRD_Year']
        gvar.sadrdYear = int(dataSettingYear[0].settingValue)
    except Exception as e:
        logging.exception('Exception occurred in ReturnTaxYear() :: ' + str(e))
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - ReturnTaxYear()', 'ReturnTaxYear', str(datetime.datetime.now())[0:23], ("Exception occurred in ReturnTaxYear() :: " + str(e)), None, 0)
        errMessage = 'E031' + "," + "" + "," + ""
        errSysMessage = dbops_obj.BuildErrorMessage(errMessage) 
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - ReturnTaxYear()', 'ReturnTaxYear', str(datetime.datetime.now())[0:23], ("Exception occurred in ReturnTaxYear() :: " + errSysMessage), None, 1)
        
    return str(gvar.sadrdYear)

@mainapp.route('/api/GetLastProcessSADRDRun', methods=['GET'])
@auth.token_required
def ReturnLastProcessSADRDRun():

    logging.debug("In ReturnLastProcessSADRDRun() method")
    gvar.lastProcessSADRDRun = ''
    try:
        gvar.sadrd_settings = dbops_obj.SadrdSysSettings()
        dataSettingLastProcessSADRDRun = [x for x in gvar.sadrd_settings if x.settingName == 'LastProcessSADRDRun']
        gvar.lastProcessSADRDRun = str(dataSettingLastProcessSADRDRun[0].settingValue)
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, "APIHome() - ReturnLastProcessSADRDRun()", 'ReturnLastProcessSADRDRun', str(datetime.datetime.now())[0:23], 'ReturnLastProcessSADRDRun method executed', None, 0)       
    except Exception as e:
        logging.exception('Exception occurred in ReturnLastProcessSADRDRun() :: ' + str(e))
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - ReturnLastProcessSADRDRun()', 'ReturnLastProcessSADRDRun', str(datetime.datetime.now())[0:23], ("Exception occurred in ReturnLastProcessSADRDRun() :: " + str(e)), None, 0)
        errMessage = 'E031' + "," + "" + "," + ""
        errSysMessage = dbops_obj.BuildErrorMessage(errMessage) 
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - ReturnLastProcessSADRDRun()', 'ReturnLastProcessSADRDRun', str(datetime.datetime.now())[0:23], ("Exception occurred in ReturnLastProcessSADRDRun() :: " + errSysMessage), None, 1)
        
    return (gvar.lastProcessSADRDRun)

@mainapp.route('/api/CloseTaxYear', methods=['POST'])
@auth.token_required
def CloseTaxYear():
    errSysMessage = ""
    logging.debug("In CloseTaxYear() method")

    response = ApihomeResp()
    gvar.sadrd_settings = dbops_obj.SadrdSysSettings()
    #gvar.sadrd_ErrMessages = dbops_obj.SADRD_Sys_Message()

    dataSettingYear = [x for x in gvar.sadrd_settings if x.settingName == 'SADRD_Year']
    reportStatus = [x for x in gvar.sadrd_settings if x.settingName == "SADRD_ReportGenerated"][0].settingValue

    if (reportStatus == 'N'):
        response.status = "Failure"
        databaseMsg = [x for x in gvar.sadrd_ErrMessages if x.MessageNumber == 'E029'][0]
        response.message = databaseMsg.Message + " " + databaseMsg.Action
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - CloseTaxYear()', 'CloseTaxYear', str(datetime.datetime.now())[0:23], response.message, None, 1)
        insertServerEventLog(mainapp.env, mainapp.config['SERVER_LOGFILES_FOLDER'], response.message, "Warning", gvar.user_id, mainapp.config['API_ENDPOINT'],
                                    gvar.func, "Close Tax Year", gvar.user_ip_address, mainapp.config['APP_SERVER_IP_ADDRESS'], "STA-025-01")
        errSysMessage = response.message
    else:
        try:
            response.message = dbops_obj.CloseTaxYear(request.data.decode(), dataSettingYear[0].settingValue)
            response.status = "Success"
            dataSettingServerFolder = [x for x in gvar.sadrd_settings if x.settingName == 'ServerFolderPath']
            dataSettingServerFolderPath =  dataSettingServerFolder[0].settingValue
            
            Inputdirpath = "Processed SA DRD"
            InputdirpathWithYear = os.path.join(Inputdirpath,dataSettingYear[0].settingValue)
            if not os.path.exists(os.path.join(dataSettingServerFolderPath,InputdirpathWithYear)):
                os.makedirs(os.path.join(dataSettingServerFolderPath,InputdirpathWithYear))
            for filename in os.listdir(dataSettingServerFolderPath):
                if filename.endswith(".xlsx") or filename.endswith(".xlsm") or filename.endswith(".csv"):
                    if dataSettingYear[0].settingValue in filename:
                        shutil.move(os.path.join(dataSettingServerFolderPath, os.path.basename(filename)), os.path.join(dataSettingServerFolderPath,os.path.join(InputdirpathWithYear, os.path.basename(filename))))
                    continue
            gvar.sadrd_settings = dbops_obj.SadrdSysSettings()
            databaseMsg = [x for x in gvar.sadrd_ErrMessages if x.MessageNumber == 'E019'][0]
            databaseMsg.Message = databaseMsg.Message.replace('[YYYY]', dataSettingYear[0].settingValue) + " " + databaseMsg.Action
            dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - CloseTaxYear()', 'CloseTaxYear', str(datetime.datetime.now())[0:23],databaseMsg.Message, None, 1)
            insertServerEventLog(mainapp.env, mainapp.config['SERVER_LOGFILES_FOLDER'], response.message, "Information", gvar.user_id, mainapp.config['API_ENDPOINT'],
                                    gvar.func, "Close Tax Year", gvar.user_ip_address, mainapp.config['APP_SERVER_IP_ADDRESS'], "STA-025-01")
            errSysMessage = response.message
        except Exception as e:
            response.status = "Failure"
            logging.exception('Exception occurred in CloseTaxYear() :: ' + str(e))
            dbops.logger.error('Exception occurred in CloseTaxYear()' + str(e))
            dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - CloseTaxYear()', 'CloseTaxYear', str(datetime.datetime.now())[0:23], ("Exception occurred in CloseTaxYear() :: " + str(e)), None, 0)
            errMessage = 'E031' + "," + "" + "," + ""
            errSysMessage = dbops_obj.BuildErrorMessage(errMessage) 
            dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - CloseTaxYear()', 'CloseTaxYear', str(datetime.datetime.now())[0:23], ("Exception occurred in CloseTaxYear() :: " + errSysMessage), None, 1)
            insertServerEventLog(mainapp.env, mainapp.config['SERVER_LOGFILES_FOLDER'], errSysMessage, "Warning", gvar.user_id, mainapp.config['API_ENDPOINT'],
                                    gvar.func, "Close Tax Year", gvar.user_ip_address, mainapp.config['APP_SERVER_IP_ADDRESS'], "STA-025-01")
    
    return jsonify({'status': response.status, 'message': errSysMessage})

@mainapp.route('/api/GetNetworkFilePath', methods=['POST'])
@auth.token_required
def ReturnNetworkFilePath():

    logging.debug("In ReturnNetworkFilePath() method")

    response = ApihomeResp()
    gvar.sadrd_settings = dbops_obj.SadrdSysSettings()

    if(AdminAuthorize(request.form)):
        try:
            for (row, sadrdSysSetting) in enumerate(gvar.sadrd_settings):
                if sadrdSysSetting.settingName == "ServerFolderPath":
                    response.status = "Success"
                    response.message = sadrdSysSetting.settingValue
                    break
        except Exception as e:
            response.status = "Failure"
            response.message = "Exception occurred in ReturnNetworkFilePath()"
            logging.exception('Exception occurred in ReturnNetworkFilePath() :: ' + str(e))
            dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - ReturnNetworkFilePath()', 'ReturnNetworkFilePath', str(datetime.datetime.now())[0:23], ("Exception occurred in ReturnNetworkFilePath() :: " + str(e)), None)
    else:
        response.status = "Unauthorized"
        response.message = "Unauthorized User. Access Denied."

    return jsonify({'status': response.status, 'message': response.message})

@mainapp.route('/api/SetDefaultFileLocation', methods=['POST'])
@auth.token_required
def SetDefaultFileLocation():
    errSysMessage = ""
    logging.debug("In SetDefaultFileLocation() method")

    response = ApihomeResp()
    gvar.sadrd_ErrMessages = dbops_obj.SADRD_Sys_Message()

    if(AdminAuthorize(request.form)):
        try:
            response.status = dbops_obj.UpdateFilePath(request.form)
            if response.status == "Success":
                databaseMsg = [x for x in gvar.sadrd_ErrMessages if x.MessageNumber == 'E021'][0]
                response.message = databaseMsg.Message.replace('[Record]', 'File Location').replace('[UserAction]', 'updated') + " " + databaseMsg.Action
            gvar.sadrd_settings = dbops_obj.SadrdSysSettings()
            dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - SetDefaultFileLocation()', 'SetDefaultFileLocation', str(datetime.datetime.now())[0:23],response.message, None, 1)
            insertServerEventLog(mainapp.env, mainapp.config['SERVER_LOGFILES_FOLDER'], response.message, "Information", gvar.user_id, mainapp.config['API_ENDPOINT'], gvar.func, "Update File Path",
                                    gvar.user_ip_address, mainapp.config['APP_SERVER_IP_ADDRESS'], "STA-025-03")
            errSysMessage = response.message
        except Exception as e:
            response.status = "Failure"
            errMessage = 'E031' + "," + "" + "," + ""
            errSysMessage = dbops_obj.BuildErrorMessage(errMessage) 
            dbops.logger.error('Exception occurred in SetDefaultFileLocation()' + str(e))
            logging.exception('Exception occurred in SetDefaultFileLocation() :: ' + str(e))
            dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - SetDefaultFileLocation()', 'SetDefaultFileLocation', str(datetime.datetime.now())[0:23], ("Exception occurred in SetDefaultFileLocation() :: " + str(e)), None, 1)
            insertServerEventLog(mainapp.env, mainapp.config['SERVER_LOGFILES_FOLDER'], errSysMessage, "Warning", gvar.user_id, mainapp.config['API_ENDPOINT'], gvar.func, "Update File Path",
                                    gvar.user_ip_address, mainapp.config['APP_SERVER_IP_ADDRESS'], "STA-025-03")
    else:
        response.status = "Unauthorized"
        errSysMessage = "Unauthorized User. Access Denied."
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - SetDefaultFileLocation()', 'SetDefaultFileLocation', str(datetime.datetime.now())[0:23], errSysMessage, None, 1)
        insertServerEventLog(mainapp.env, mainapp.config['SERVER_LOGFILES_FOLDER'], errSysMessage, "Warning", gvar.user_id, mainapp.config['API_ENDPOINT'], gvar.func, "Update File Path",
                                gvar.user_ip_address, mainapp.config['APP_SERVER_IP_ADDRESS'], "STA-025-03")

    return jsonify({'status': response.status, 'message': errSysMessage})

@mainapp.route('/api/GetSettingsData', methods=['GET'])
@auth.token_required
def GetSettingsData():

    logging.debug("In GetSettingsData() method")
    
    dictSettingsList = []

    try:
        gvar.sadrd_settings = dbops_obj.SadrdSysSettings()
        for (row, setting) in enumerate(gvar.sadrd_settings):
            if setting.ShowInUI:
                dictSetting = dict([('settingName', setting.settingName), ('settingValue', setting.settingValue), ('ShowInUI', setting.ShowInUI), ('Description', setting.Description), ('DataType', setting.DataType)])
                dictSettingsList.append(dictSetting)
    except Exception as e:
        logging.exception('Exception occurred in GetSettingsData() :: ' + str(e))
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - GetSettingsData()', 'GetSettingsData', str(datetime.datetime.now())[0:23], ("Exception occurred in GetSettingsData() :: " + str(e)), None, 1)
        dbops.logger.error('Exception occurred in GetSettingsData()' + str(e))

    return jsonify({'SettingsData' : dictSettingsList})

@mainapp.route('/api/UpdateSettingsData', methods=['POST'])
@auth.token_required
def UpdateSettingsData():
    errSysMessage = ""
    logging.debug("In UpdateSettingsData() method")
    
    settingsAddOrUpdate = ''
    response = ApihomeResp()
    gvar.sadrd_ErrMessages = dbops_obj.SADRD_Sys_Message()

    try:
        gvar.sadrd_settings = dbops_obj.SadrdSysSettings()
        response.status = dbops_obj.UpdateSettingsData(request.form)
        if response.status == "Exists":
            settingsAddOrUpdate = 'Update Setting'
            databaseMsg = [x for x in gvar.sadrd_ErrMessages if x.MessageNumber == 'E023'][0]
            keyValueMsg = "Setting Name - " + request.form['settingName'] + " and Setting Value - " + request.form['settingValue']
            response.message = databaseMsg.Message.replace('[insert key-values here]', keyValueMsg) + " " + databaseMsg.Action
            errSysMessage = response.message
        else:
            databaseMsg = [x for x in gvar.sadrd_ErrMessages if x.MessageNumber == 'E021'][0]
            if ("add" == request.form['userAction']):
                settingsAddOrUpdate = 'Add Setting'
                response.message = databaseMsg.Message.replace('[Record]', request.form['settingName']).replace('[UserAction]', 'updated') + " " + databaseMsg.Action
            else:
                settingsAddOrUpdate = 'Update Setting'
                response.message = databaseMsg.Message.replace('[Record]', request.form['settingName']).replace('[UserAction]', 'updated') + " " + databaseMsg.Action   
            errSysMessage = response.message
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, "APIHome - UpdateSettingsData()", settingsAddOrUpdate, str(datetime.datetime.now())[0:23],response.message, None, 1)   
        insertServerEventLog(mainapp.env, mainapp.config['SERVER_LOGFILES_FOLDER'], response.message, "Information", gvar.user_id, mainapp.config['API_ENDPOINT'],
                            gvar.func, settingsAddOrUpdate, gvar.user_ip_address, mainapp.config['APP_SERVER_IP_ADDRESS'], "STA-025-03") 
        
    except Exception as e:
        response.status = "Failure"
        dbops.logger.error('Exception occurred in UpdateSettingsData() :: ' + str(e))
        logging.exception('Exception occurred in UpdateSettingsData() :: ' + str(e))
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome - UpdateSettingsData()', 'Setting Add/Update', str(datetime.datetime.now())[0:23], 'Exception occurred in UpdateSettingsData() :: ' + str(e), None,0)
        errMessage = 'E031' + "," + "" + "," + ""
        errSysMessage = dbops_obj.BuildErrorMessage(errMessage) 
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome - UpdateSettingsData()', 'Setting Add/Update', str(datetime.datetime.now())[0:23], 'Exception occurred in UpdateSettingsData() :: ' + errSysMessage, None,1)
        insertServerEventLog(mainapp.env, mainapp.config['SERVER_LOGFILES_FOLDER'], errSysMessage, "Warning", gvar.user_id, mainapp.config['API_ENDPOINT'],
                            gvar.func, settingsAddOrUpdate, gvar.user_ip_address, mainapp.config['APP_SERVER_IP_ADDRESS'], "STA-025-03") 
        
    return jsonify({'status': response.status, 'message': errSysMessage})
 
@mainapp.route('/api/GetAdmScheduleEData', methods=['GET'])
@auth.token_required
def GetAdmScheduleEData():

    logging.debug("In GetAdmScheduleEData() method")

    dictSchEList = []

    try:
        gvar.scheduleEList = dbops_obj.SadrdAdmScheduleE()
        for (row, schERec) in enumerate(gvar.scheduleEList):
            dictSchE = dict([('Cusip', schERec.Cusip), ('CusipName', schERec.CusipName)])
            dictSchEList.append(dictSchE)
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, "APIHome() - GetAdmScheduleEData()", 'GetAdmScheduleEData', str(datetime.datetime.now())[0:23], 'GetAdmScheduleEData method executed', None, 0)       
    except Exception as e:
        logging.exception('Exception occurred in GetAdmScheduleEData() :: ' + str(e))
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - GetAdmScheduleEData()', 'GetAdmScheduleEData', str(datetime.datetime.now())[0:23], ("Exception occurred in GetAdmScheduleEData() :: " + str(e)), None, 0)
        errMessage = 'E031' + "," + "" + "," + ""
        errSysMessage = dbops_obj.BuildErrorMessage(errMessage) 
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - GetAdmScheduleEData()', 'GetAdmScheduleEData', str(datetime.datetime.now())[0:23], ("Exception occurred in GetAdmScheduleEData() :: " + errSysMessage), None, 1)
        
        dbops.logger.error('Exception occurred in GetScheduleEData()' + str(e))
        
    return jsonify({'ScheduleEData' : dictSchEList})

@mainapp.route('/api/UpdateAdmScheduleEData', methods=['POST'])
@auth.token_required
def UpdateAdmScheduleEData():
    errSysMessage = ""
    logging.debug("In UpdateAdmScheduleEData() method")

    schEAddOrUpdate = ''
    response = ApihomeResp()
    gvar.sadrd_ErrMessages = dbops_obj.SADRD_Sys_Message()
    
    try:
        response.status = dbops_obj.UpdateAdmScheduleEData(request.form)
        if response.status == "Exists":
            schEAddOrUpdate = "Update Sch E Cusip"
            databaseMsg = [x for x in gvar.sadrd_ErrMessages if x.MessageNumber == 'E023'][0]
            keyValueMsg = "Cusip - " + request.form['Cusip']
            response.message = databaseMsg.Message.replace('[insert key-values here]', keyValueMsg) + " " + databaseMsg.Action
        else:
            databaseMsg = [x for x in gvar.sadrd_ErrMessages if x.MessageNumber == 'E021'][0]
            if ("add" == request.form['userAction']):
                schEAddOrUpdate = "Add Sch E Cusip"
                response.message = databaseMsg.Message.replace('[Record]', request.form['Cusip']).replace('[UserAction]', 'added') + " " + databaseMsg.Action
            elif ("update" == request.form['userAction']):
                schEAddOrUpdate = "Update Sch E Cusip"
                response.message = databaseMsg.Message.replace('[Record]', request.form['Cusip']).replace('[UserAction]', 'updated') + " " + databaseMsg.Action
            else:
                schEAddOrUpdate = "Delete Sch E Cusip"
                response.message = databaseMsg.Message.replace('[Record]', request.form['Cusip']).replace('[UserAction]', 'deleted') + " " + databaseMsg.Action
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, "APIHome() - UpdateAdmScheduleEData()", schEAddOrUpdate, str(datetime.datetime.now())[0:23], response.message, None, 1)   
        insertServerEventLog(mainapp.env, mainapp.config['SERVER_LOGFILES_FOLDER'], response.message, "Information", gvar.user_id, mainapp.config['API_ENDPOINT'],
                                gvar.func, schEAddOrUpdate, gvar.user_ip_address, mainapp.config['APP_SERVER_IP_ADDRESS'], "STA-025-03")
        errSysMessage = response.message
    except Exception as e:
        response.status = "Failure"
        if ("add" == request.form['userAction']):
            response.message = "Exception occurred while adding the record."
        elif ("update" == request.form['userAction']):
            response.message = "Exception occurred while updating the record."
        else:
            response.message = "Exception occurred while deleting the record."
        logging.exception('Exception occurred in UpdateAdmScheduleEData() :: ' + str(e))
        dbops.logger.error('Exception occurred in UpdateAdmScheduleEData() :: ' + str(e))
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - UpdateAdmScheduleEData()', 'Sch E Cusip Add/Update', str(datetime.datetime.now())[0:23], ('Exception occurred in UpdateAdmScheduleEData() :: ' + str(e)), None, 0)
        errMessage = 'E031' + "," + "" + "," + ""
        errSysMessage = dbops_obj.BuildErrorMessage(errMessage) 
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - UpdateAdmScheduleEData()', 'Sch E Cusip Add/Update', str(datetime.datetime.now())[0:23], ('Exception occurred in UpdateAdmScheduleEData() :: ' + errSysMessage), None, 1)
        insertServerEventLog(mainapp.env, mainapp.config['SERVER_LOGFILES_FOLDER'], response.message, "Warning", gvar.user_id, mainapp.config['API_ENDPOINT'],
                               gvar.func, schEAddOrUpdate, gvar.user_ip_address, mainapp.config['APP_SERVER_IP_ADDRESS'], "STA-025-03")  
         
    return jsonify({'status': response.status, 'message': errSysMessage})

@mainapp.route('/api/GetHelpTextMessage', methods=['POST'])
@auth.token_required
def GetHelpTextMessage():
    logging.debug("In GetHelpTextMessage() method")

    messageCodeList = []
    helpTextMsgOutput = []

    gvar.sadrd_ErrMessages = dbops_obj.SADRD_Sys_Message()
    inputArgsList = json.loads(request.form['HelpTextInput'])

    for x in inputArgsList:
        messageCodeList.append(x['messageCode'])
    
    for messageCode in messageCodeList:
        for x in gvar.sadrd_ErrMessages:
            if x.MessageNumber == messageCode:
                helpTextMsgOutput.append({"messageCode" : messageCode, "message" : x.Message, "action" : x.Action})
    
    dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, "Import", 'APIHome() - GetHelpTextMessage()', str(datetime.datetime.now())[0:23], 'GetHelpTextMessage method executed', None, 0)       
    return jsonify(helpTextMsgOutput)

@mainapp.route('/api/GetQualPctData', methods=['GET'])
@auth.token_required
def GetQualPctData():

    logging.debug("In GetQualPctData() method") 

    dictQualPctList = []

    try:
        gvar.qualPctList = dbops_obj.GetQualPctOverrideData()
        for (row, qualPctRec) in enumerate(gvar.qualPctList):
            dictqualPct = dict([('Year', qualPctRec.Year), ('Company', qualPctRec.Company), ('Cusip', qualPctRec.Cusip), ('QualPct', (f'{(qualPctRec.QualPct * 100):.8f}'))])
            dictQualPctList.append(dictqualPct)
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, "APIHome() - GetQualPctData()", 'GetQualPctData', str(datetime.datetime.now())[0:23], 'GetQualPctData method executed', None, 0)       
    except Exception as e:
        logging.exception('Exception occurred in GetQualPctData() :: ' + str(e))
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - GetQualPctData()', 'GetQualPctData', str(datetime.datetime.now())[0:23], ("Exception occurred in GetQualPctData() :: " + str(e)), None)
        dbops.logger.error('Exception occurred in GetQualPctData()' + str(e))

    return jsonify({'QualPctData' : dictQualPctList})

@mainapp.route('/api/GetValidCompanyList', methods=['GET'])
@auth.token_required
def GetValidCompanyList():

    logging.debug("In GetCompanyList() method") 

    dictCompanyList = []
    
    try:
        gvar.sadrd_settings = dbops_obj.SadrdSysSettings()
        for (row, sadrdSysSetting) in enumerate(gvar.sadrd_settings):
            if sadrdSysSetting.settingName == "Valid_Company":
                dictCompanyList.append(sadrdSysSetting.settingValue)
                continue
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, "APIHome() - GetValidCompanyList()", 'GetValidCompanyList', str(datetime.datetime.now())[0:23], 'GetValidCompanyList method executed', None, 0)
    except Exception as e:
        logging.exception('Exception occurred in GetValidCompanyList() :: ' + str(e))
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - GetValidCompanyList()', 'GetValidCompanyList', str(datetime.datetime.now())[0:23], ("Exception occurred in GetValidCompanyList() :: " + str(e)), None)

    return jsonify({'CompanyList' : dictCompanyList})

@mainapp.route('/api/UpdateQualPctData', methods=['POST'])
@auth.token_required
def UpdateQualPctData():
    errSysMessage = ""
    logging.debug("In UpdateQualPctData() method")

    qualPctAddOrUpdate = ''
    response = ApihomeResp()
    gvar.sadrd_ErrMessages = dbops_obj.SADRD_Sys_Message()

    try:
        gvar.qualPctList = dbops_obj.GetQualPctOverrideData()
        response.status = dbops_obj.UpdateQualPctData(request.form)
        if response.status == "Exists":
            qualPctAddOrUpdate = "Qual % Override Updated"
            databaseMsg = [x for x in gvar.sadrd_ErrMessages if x.MessageNumber == 'E023'][0]
            keyValueMsg = "Year - " + request.form['Year'] + ", Company - " + request.form['Company'] + ", and Cusip - " + request.form['Cusip']
            response.message = databaseMsg.Message.replace('[insert key-values here]', keyValueMsg) + " " + databaseMsg.Action
        else:
            databaseMsg = [x for x in gvar.sadrd_ErrMessages if x.MessageNumber == 'E021'][0]
            if ("add" == request.form['userAction']):
                qualPctAddOrUpdate = "Add Qual % Override"
                response.message = databaseMsg.Message.replace('[Record]', 'Qual % Override').replace('[UserAction]', 'added') + " " + databaseMsg.Action
            elif ("update" == request.form['userAction']):
                qualPctAddOrUpdate = "Update Qual % Override"
                response.message = databaseMsg.Message.replace('[Record]', 'Qual % Override').replace('[UserAction]', 'updated') + " " + databaseMsg.Action
            else:
                qualPctAddOrUpdate = "Delete Qual % Override"
                response.message = databaseMsg.Message.replace('[Record]', request.form['Cusip']).replace('[UserAction]', 'deleted') + " " + databaseMsg.Action
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, "APIHome() - UpdateQualPctData()", qualPctAddOrUpdate, str(datetime.datetime.now())[0:23], response.message, None, 1)
        insertServerEventLog(mainapp.env, mainapp.config['SERVER_LOGFILES_FOLDER'], response.message, "Information", gvar.user_id, mainapp.config['API_ENDPOINT'],
                                gvar.func, qualPctAddOrUpdate, gvar.user_ip_address, mainapp.config['APP_SERVER_IP_ADDRESS'], "STA-025-03")
        errSysMessage = response.message
    except Exception as e:
        response.status = "Failure"
        if ("add" == request.form['userAction']):
            response.message = "Exception occurred while adding the record."
        elif ("update" == request.form['userAction']):
            response.message = "Exception occurred while updating the record."
        else:
            response.message = "Exception occurred while deleting the record."
        dbops.logger.error('Exception occurred in UpdateQualPctData() :: ' + str(e))
        logging.exception('Exception occurred in UpdateQualPctData() :: ' + str(e))
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - UpdateQualPctData()', 'Qual % Override Add/Update', str(datetime.datetime.now())[0:23], ("Exception occurred in UpdateQualPctData() :: " + str(e)), None, 0)
        errMessage = 'E031' + "," + "" + "," + ""
        errSysMessage = dbops_obj.BuildErrorMessage(errMessage) 
        dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'APIHome() - UpdateQualPctData()', 'Qual % Override Add/Update', str(datetime.datetime.now())[0:23], ("Exception occurred in UpdateQualPctData() :: " + errSysMessage), None, 1)
        
        insertServerEventLog(mainapp.env, mainapp.config['SERVER_LOGFILES_FOLDER'], response.message, "Warning", gvar.user_id, mainapp.config['API_ENDPOINT'],
                                gvar.func, qualPctAddOrUpdate, gvar.user_ip_address, mainapp.config['APP_SERVER_IP_ADDRESS'], "STA-025-03")
    
    return jsonify({'status': response.status, 'message': errSysMessage})

def AdminAuthorize(inputRequest):
    
    logging.debug("In AdminAuthorize() method")

    roleType = ''
    authorized = False

    try:
        sadrdUsersList = dbops_obj.GetAllUsers()
        if len(sadrdUsersList):
            rolesList = GetAllRoles().json.get('RolesData')
            datatb = [x for x in sadrdUsersList if x.NetworkId == inputRequest['NetworkId']]
            roleType = [x for x in rolesList if x['RoleID'] == datatb[0].RoleId][0]['Type']
            authorized = (len(datatb) and roleType == inputRequest['Role'] and str(datatb[0].RoleId) == inputRequest['RoleId'] and roleType == 'Admin')
            insertServerEventLog(mainapp.env, mainapp.config['SERVER_LOGFILES_FOLDER'], ("User is Admin" if authorized else "User is not Admin"), "Information", gvar.user_id,
                                    mainapp.config['API_ENDPOINT'], gvar.func, "Admin Authorization", gvar.user_ip_address, mainapp.config['APP_SERVER_IP_ADDRESS'], "STA-025-06")
    except Exception as e:
        dbops.logger.error('Exception occurred in AdminAuthorize() :: ' + str(e))
        logging.exception('Exception occurred in AdminAuthorize() :: ' + str(e))

    dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'AdminAuthorize()', 'AdminAuthorize', str(datetime.datetime.now())[0:23], 'AdminAuthorize task', None,0)
    
    return authorized

if __name__ == "__main__":
    mainapp.run()
