import datetime
import logging
#from cryptography.fernet import Fernet
import jwt
#import msal
from flask import request, jsonify
from functools import wraps
import globalvars as gvar
import Services.dboperations as dbops
import os

# def decrypt(token: bytes, key: bytes) -> bytes:
#     return Fernet(key).decrypt(token)

# def encrypt(message: bytes, key: bytes) -> bytes:
#     return Fernet(key).encrypt(message)

def DecryptToken(token):
    logging.debug("In DecryptToken() method")
    data = str(token)[7:]       # Strip leading "Bearer "
    data = data.encode('utf-8') # Converting the token in bytes
    decoded_token = jwt.decode(data, options={"verify_signature": False})
    return decoded_token

# def GetBearerToken():
#     print("In GetBearerToken")
#     apirespmsg = ""
#     token = ""

#     try:
#         config = dict()
#         config["client_id"] = '72d318f6-aae0-42ed-aa49-07040c48c704'        # client id for Manulife
#         config["authority"] = 'https://login.microsoftonline.com/5d3e2773-e07f-4432-a630-1a0f68a28a05'  # Microsoft's AAD (Azure Active Directory) authorization site
#         config["scope"]     = ['User.Read', 'User.Read.All']
#         cache = msal.SerializableTokenCache()
#         #to do
#         #gvar.user_id = str(os.getlogin())
#         #cache_filepath = "C:/Users/" + gvar.user_id + "/SADRD" + "/my_cache.bin"
#         # cache_path = str(os.environ._data['APPDATA']).replace("\\",'/') + "/SADRD"
#         cache_path = gvar.gconfig['SADRD_CACHE_FOLDER'].replace("\\",'/')
#         assure_folder_exists(cache_path)
#         cache_filepath = cache_path  + "/my_cache.bin"
#         if os.path.exists(cache_filepath):
#             cache.deserialize(open(cache_filepath, "r").read())
        
#         public_client_app = msal.PublicClientApplication(config["client_id"],authority=config["authority"],token_cache=cache)
#         result = None
#         accounts = public_client_app.get_accounts()
#         if accounts:
#             # Choose the first account listed for this user, e.g. sinvija@MFCGD.COM
#             chosen_account = accounts[0]
#             # Now let's try to find a token for this account in the user's cache.
#             result = public_client_app.acquire_token_silent(config["scope"], account=chosen_account)
        
#         if not result:
#             result = public_client_app.acquire_token_interactive(config["scope"])                        
#             open(cache_filepath, "w").write(cache.serialize())

#         if "access_token" in result:
#             # Encrypt the token.
#             key = "VeabYVl4KfOKFMYguyB-cjxLl7Wg4u9sayX0AXoUQFM=".encode()
#             encrypted_token = encrypt(result['access_token'].encode(), key)
#             encrypted_token_str = encrypted_token.decode('utf-8')
#             token = encrypted_token_str
#             apirespmsg = "isAdAuthenticated"
#         else:
#             apirespmsg = 'Authorization header error: ' + str(result.get("error")) + ' - ' + str(result.get("error_description")) + \
#                 '. Correlation id: ' + str(result.get("correlation_id"))  # You may need correlation_id when reporting a bug

#     except Exception as e:
#         #logger.error ('API Home: Error in processing GetBearerToken request: ' + str(e))
#         apirespmsg = 'Exception occurred in API function ' + gvar.func

#     return jsonify({'apirespmsg':apirespmsg, 'Token': token})

def GetLoggedInUser(token):
    logging.debug("In GetLoggedInUser() method")
    decrypted_token = DecryptToken(token)
    loggedInUser = (str(decrypted_token['unique_name']))
    if loggedInUser.find('@MFCGD.COM') > 0:
        gvar.user_id = loggedInUser[0:loggedInUser.find('@MFCGD.COM')]
    else:
        gvar.user_id = loggedInUser
    return gvar.user_id

def token_required(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        gvar.func = f.__name__
        gvar.user_id = ''
        gvar.user_name = ''
        gvar.user_id_short = ''
        gvar.user_ip_address = ''

        logging.debug("In token_required() method :: " + gvar.func)

        if (request.headers.__contains__('Authorization')):
            data = request.headers['Authorization']
            try:
                decrypted_token = DecryptToken(data)    
                gvar.user_id_short = GetLoggedInUser(data)
                gvar.user_name = str(decrypted_token['name'])
                gvar.user_ip_address = str(decrypted_token['ipaddr'])
                
                if len(gvar.sadrdUsersList):
                    datatb = [x for x in gvar.sadrdUsersList if x.NetworkId == gvar.user_id_short]
                    if len(datatb):
                        gvar.ISAUTHORIZED = True
                        logging.debug("In token_required() method :: User is Authorized")
                    else:
                        gvar.ISAUTHORIZED = False
                        logging.debug("In token_required() method :: Unauthorized Access")
                        return jsonify({'apirespmsg': 'Unauthorized Access'})
                else:
                    gvar.ISAUTHORIZED = False
                    logging.debug("In token_required() method :: Unauthorized Access")
                    return jsonify({'apirespmsg': 'Unauthorized Access'})
            except Exception as e:
                dbops_obj = dbops.dboperations()
                dbops_obj.insert_actionLog(datetime.datetime.now().month, datetime.datetime.now().year, gvar.user_id, 'token_required', 'token_required', str(datetime.datetime.now())[0:23], ("Exception occured in token_required() :: " + str(e)), None)
                logging.exception("Exception occurred in token_required() method :: " + str(e))
                return jsonify({'apirespmsg' : 'Invalid Token!'}), 494
            return f(*args, **kwargs)
        else:
            gvar.ISAUTHORIZED = False
            logging.debug("In token_required() method :: Authorization Token is missing!")
            return jsonify({'apirespmsg': 'Authorization Token is missing!'})

    return decorated

# def assure_folder_exists(folder):
#     logging.debug("In assure_folder_exists() method")    
#     if not os.path.exists(folder):
#         os.makedirs(folder)
