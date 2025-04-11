06:39:03 Start computing revisions
06:39:04 Finalize build settings
06:43:53 The build is removed from the queue to be prepared for the start
06:43:53 Starting the build on the agent "aks-teamcity-python3-9-1543"
06:43:54 Updating tools for build
06:43:54 Clearing temporary directory: /opt/buildagent/temp/buildTmp
06:43:55 Retrieved 0 secrets from Azure Key Vault
06:43:55 Publishing internal artifacts
06:43:55 Clean build enabled: removing old files from /opt/buildagent/work/9422c6f94111a311
06:43:55 Checkout directory: /opt/buildagent/work/9422c6f94111a311
06:43:55 Updating sources: auto checkout (on server)
06:43:57 Step 1/16: PYTHON AND PIP VERSION (Command Line)
06:43:59 Step 2/16: CREATE VIRTUAL ENV : WINDOWS (PowerShell)
06:43:59 Step 3/16: CREATE VIRTUAL ENV : LINUX (Command Line)
06:43:59 Step 4/16: PIP INSTALL : WINDOWS (PowerShell)
06:43:59 Step 5/16: PIP INSTALL : LINUX (Command Line)
06:46:38 Step 6/16: Install ODBC 17 (Command Line)
06:46:49 Step 7/16: PYTHON PYTEST : LINUX (Command Line)
06:46:49   Starting: /opt/buildagent/temp/agentTmp/custom_script11531409329032587863
06:46:49   in directory: /opt/buildagent/work/9422c6f94111a311
06:46:49   pip install required modules for pytest..
06:46:49   checking OBDC Driver...
06:46:49   [ODBC Driver 17 for SQL Server]
06:46:49   Finding libmsodbcsql...
06:46:50   /usr/lib/libmsodbcsql-17.so
06:46:50   /usr/lib64/libmsodbcsql-17.so
06:46:50   /opt/microsoft/msodbcsql17/lib64/libmsodbcsql-17.10.so.6.1
06:46:50   Running test...
06:46:52   ============================= test session starts ==============================
06:46:52   platform linux -- Python 3.9.5, pytest-7.2.0, pluggy-1.5.0
06:46:52   rootdir: /opt/buildagent/work/9422c6f94111a311
06:46:52   plugins: cov-4.0.0, Flask-Dance-3.2.0
06:46:52   collected 89 items
06:46:52   
06:46:52   test/test_APIHome.py .........                                           [ 10%]
06:46:52   test/test_config.py .....................                                [ 33%]
06:46:52   test/test_db.py .                                                        [ 34%]
06:46:52   test/test_globalvars.py .                                                [ 35%]
06:46:52   test/Entities/test_Customentities.py ....                                [ 40%]
06:46:52   test/Entities/test_dbormschemas.py .........                             [ 50%]
06:46:52   test/Services/test_APIResponse.py .......                                [ 58%]
06:46:52   test/Services/test_CustomException.py ..                                 [ 60%]
06:46:52   test/Services/test_dboperations.py ...........                           [ 73%]
06:46:53   test/Services/test_fileoperations.py .FF.FFFFFFFF..                      [ 88%]
06:46:53   test/Services/test_logoperations.py ...                                  [ 92%]
06:46:53   test/Services/test_parentparser.py .......                               [100%]
06:46:53   
06:46:53   =================================== FAILURES ===================================
06:46:53   ___ Test_FileOperations.test_Downloadfilenames_toprocess_annual_schd_success ___
06:46:53   
06:46:53   self = <test.Services.test_fileoperations.Test_FileOperations testMethod=test_Downloadfilenames_toprocess_annual_schd_success>
06:46:53   
06:46:53       def test_Downloadfilenames_toprocess_annual_schd_success(self):
06:46:53           server_path, local_path, action, year = 's_path', 'l_path', 'Annual Stmt - Sch D', '2023'
06:46:53           filename = '2023JHUSASchD.csv'
06:46:53           self.mock_listdir.return_value = [filename]
06:46:53           result = fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
06:46:53           expected_dest_path = self._build_path(local_path, filename)
06:46:53           expected_src_path = self._build_path(server_path, filename)
06:46:53           self.assertEqual(result, [expected_dest_path])
06:46:53   >       self.assertPathCalledWith(self.mock_is_open, (expected_src_path,))
06:46:53   
06:46:53   test/Services/test_fileoperations.py:165:
06:46:53   _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
06:46:53   test/Services/test_fileoperations.py:96: in assertPathCalledWith
06:46:53       self.assertEqual(normalized_actual, normalized_expected,
06:46:53   E   AssertionError: Tuples differ: ('s_path\\2023JHUSASchD.csv',) != ('s_path/2023JHUSASchD.csv',)
06:46:53   E
06:46:53   E   First differing element 0:
06:46:53   E   's_path\\2023JHUSASchD.csv'
06:46:53   E   's_path/2023JHUSASchD.csv'
06:46:53   E
06:46:53   E   - ('s_path\\2023JHUSASchD.csv',)
06:46:53   E   ?         ^^
06:46:53   E
06:46:53   E   + ('s_path/2023JHUSASchD.csv',)
06:46:53   E   ?         ^
06:46:53   E    : Normalized call args differ for 'is_open':
06:46:53   E     Actual: ('s_path\\2023JHUSASchD.csv',)
06:46:53   E   Expected: ('s_path/2023JHUSASchD.csv',)
06:46:53   _ Test_FileOperations.test_Downloadfilenames_toprocess_dest_dir_does_not_exist _
06:46:53   
06:46:53   self = <test.Services.test_fileoperations.Test_FileOperations testMethod=test_Downloadfilenames_toprocess_dest_dir_does_not_exist>
06:46:53   
06:46:53       def test_Downloadfilenames_toprocess_dest_dir_does_not_exist(self):
06:46:53           server_path, local_path, action, year = 's_path', 'l_path_new', 'Annual Stmt - Sch D', '2023'
06:46:53           filename = '2023JHUSASchD.csv'
06:46:53           self.mock_exists.side_effect = lambda p: False if p == local_path else True
06:46:53           self.mock_listdir.return_value = [filename]
06:46:53           fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
06:46:53           self.mock_exists.assert_any_call(local_path)
06:46:53           self.mock_makedirs.assert_called_once_with(local_path)
06:46:53           expected_src_path = self._build_path(server_path, filename)
06:46:53           expected_dest_path = self._build_path(local_path, filename)
06:46:53   >       self.assertPathCalledWith(self.mock_is_open, (expected_src_path,))
06:46:53   
06:46:53   test/Services/test_fileoperations.py:264:
06:46:53   _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
06:46:53   test/Services/test_fileoperations.py:96: in assertPathCalledWith
06:46:53       self.assertEqual(normalized_actual, normalized_expected,
06:46:53   E   AssertionError: Tuples differ: ('s_path\\2023JHUSASchD.csv',) != ('s_path/2023JHUSASchD.csv',)
06:46:53   E
06:46:53   E   First differing element 0:
06:46:53   E   's_path\\2023JHUSASchD.csv'
06:46:53   E   's_path/2023JHUSASchD.csv'
06:46:53   E
06:46:53   E   - ('s_path\\2023JHUSASchD.csv',)
06:46:53   E   ?         ^^
06:46:53   E
06:46:53   E   + ('s_path/2023JHUSASchD.csv',)
06:46:53   E   ?         ^
06:46:53   E    : Normalized call args differ for 'is_open':
06:46:54   E     Actual: ('s_path\\2023JHUSASchD.csv',)
06:46:54   E   Expected: ('s_path/2023JHUSASchD.csv',)
06:46:54   ________ Test_FileOperations.test_Downloadfilenames_toprocess_file_open ________
06:46:54   
06:46:54   self = <test.Services.test_fileoperations.Test_FileOperations testMethod=test_Downloadfilenames_toprocess_file_open>
06:46:54   
06:46:54       def test_Downloadfilenames_toprocess_file_open(self):
06:46:54           server_path, local_path, action, year = 's_path', 'l_path', 'Annual Stmt - Sch D', '2023'
06:46:54           filename = '2023JHUSASchD.csv'
06:46:54           self.mock_listdir.return_value = [filename]
06:46:54           self.mock_is_open.return_value = True
06:46:54           with self.assertRaises(FileValidationException):
06:46:54               fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
06:46:54           expected_src_path = self._build_path(server_path, filename)
06:46:54   >       self.assertPathCalledWith(self.mock_is_open, (expected_src_path,))
06:46:54   
06:46:54   test/Services/test_fileoperations.py:231:
06:46:54   _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
06:46:54   test/Services/test_fileoperations.py:96: in assertPathCalledWith
06:46:54       self.assertEqual(normalized_actual, normalized_expected,
06:46:54   E   AssertionError: Tuples differ: ('s_path\\2023JHUSASchD.csv',) != ('s_path/2023JHUSASchD.csv',)
06:46:54   E
06:46:54   E   First differing element 0:
06:46:54   E   's_path\\2023JHUSASchD.csv'
06:46:54   E   's_path/2023JHUSASchD.csv'
06:46:54   E
06:46:54   E   - ('s_path\\2023JHUSASchD.csv',)
06:46:54   E   ?         ^^
06:46:54   E
06:46:54   E   + ('s_path/2023JHUSASchD.csv',)
06:46:54   E   ?         ^
06:46:54   E    : Normalized call args differ for 'is_open':
06:46:54   E     Actual: ('s_path\\2023JHUSASchD.csv',)
06:46:54   E   Expected: ('s_path/2023JHUSASchD.csv',)
06:46:54   ----------------------------- Captured stderr call -----------------------------
06:46:54   Services.dboperations: ERROR    Error in Downloadfilenames_toprocess -Mocked Error Message
06:46:54   root        : ERROR    Exception occurred in Downloadfilenames_toprocess() method :: Mocked Error Message
06:46:54   Traceback (most recent call last):
06:46:54     File "/opt/buildagent/work/9422c6f94111a311/Services/fileoperations.py", line 146, in Downloadfilenames_toprocess
06:46:54       raise CustomException.FileValidationException(errMessageComplete)
06:46:54   Services.CustomException.FileValidationException: Mocked Error Message
06:46:54   ------------------------------ Captured log call -------------------------------
06:46:54   ERROR    Services.dboperations:fileoperations.py:148 Error in Downloadfilenames_toprocess -Mocked Error Message
06:46:54   ERROR    root:fileoperations.py:149 Exception occurred in Downloadfilenames_toprocess() method :: Mocked Error Message
06:46:54   Traceback (most recent call last):
06:46:54     File "/opt/buildagent/work/9422c6f94111a311/Services/fileoperations.py", line 146, in Downloadfilenames_toprocess
06:46:54       raise CustomException.FileValidationException(errMessageComplete)
06:46:54   Services.CustomException.FileValidationException: Mocked Error Message
06:46:54   ___ Test_FileOperations.test_Downloadfilenames_toprocess_ftcgrossup_success ____
06:46:54   
06:46:54   self = <test.Services.test_fileoperations.Test_FileOperations testMethod=test_Downloadfilenames_toprocess_ftcgrossup_success>
06:46:54   
06:46:54       def test_Downloadfilenames_toprocess_ftcgrossup_success(self):
06:46:54           server_path, local_path, action, year = 's_path', 'l_path', 'FTCGrossup', '2023'
06:46:54           files = [f'2023_Q{i}_FTCGrossup.xlsx' for i in range(1, 5)]
06:46:54           self.mock_listdir.return_value = files
06:46:54           result = fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
06:46:54           expected_dest_files = [self._build_path(local_path, f) for f in files]
06:46:54           expected_src_files = [self._build_path(server_path, f) for f in files]
06:46:54           self.assertEqual(sorted(result), sorted(expected_dest_files))
06:46:54           self.assertEqual(self.mock_is_open.call_count, 4, "is_open call count mismatch")
06:46:54           try:
06:46:54               actual_is_open_args = {os.path.normpath(c.args[0]) for c in self.mock_is_open.call_args_list}
06:46:54           except (AttributeError, IndexError, TypeError) as e:
06:46:54                raise AssertionError(f"Could not extract args from mock_is_open.call_args_list: {e}\nList: {self.mock_is_open.call_args_list}")
06:46:54           expected_is_open_args = {os.path.normpath(p) for p in expected_src_files}
06:46:54   >       self.assertSetEqual(actual_is_open_args, expected_is_open_args, "is_open call arguments mismatch")
06:46:54   E       AssertionError: Items in the first set but not the second:
06:46:54   E       's_path\\2023_Q1_FTCGrossup.xlsx'
06:46:54   E       's_path\\2023_Q2_FTCGrossup.xlsx'
06:46:54   E       's_path\\2023_Q3_FTCGrossup.xlsx'
06:46:54   E       's_path\\2023_Q4_FTCGrossup.xlsx'
06:46:54   E       Items in the second set but not the first:
06:46:54   E       's_path/2023_Q1_FTCGrossup.xlsx'
06:46:54   E       's_path/2023_Q4_FTCGrossup.xlsx'
06:46:54   E       's_path/2023_Q2_FTCGrossup.xlsx'
06:46:54   E       's_path/2023_Q3_FTCGrossup.xlsx' : is_open call arguments mismatch
06:46:54   
06:46:54   test/Services/test_fileoperations.py:195: AssertionError
06:46:54   ___ Test_FileOperations.test_Downloadfilenames_toprocess_ignores_temp_files ____
06:46:54   
06:46:54   self = <test.Services.test_fileoperations.Test_FileOperations testMethod=test_Downloadfilenames_toprocess_ignores_temp_files>
06:46:54   
06:46:54       def test_Downloadfilenames_toprocess_ignores_temp_files(self):
06:46:54           server_path, local_path, action, year = 's_path', 'l_path', 'Annual Stmt - Sch D', '2023'
06:46:54           good_file, temp_file = '2023JHUSASchD.csv', '~$2023JHUSASchD.csv'
06:46:54           self.mock_listdir.return_value = [temp_file, good_file]
06:46:54           result = fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
06:46:54           expected_dest_path = self._build_path(local_path, good_file)
06:46:54           expected_src_path = self._build_path(server_path, good_file)
06:46:54           self.assertEqual(result, [expected_dest_path])
06:46:54   >       self.assertPathCalledWith(self.mock_is_open, (expected_src_path,))
06:46:54   test/Services/test_fileoperations.py:276:
06:46:54   _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
06:46:54   test/Services/test_fileoperations.py:96: in assertPathCalledWith
06:46:54       self.assertEqual(normalized_actual, normalized_expected,
06:46:54   E   AssertionError: Tuples differ: ('s_path\\2023JHUSASchD.csv',) != ('s_path/2023JHUSASchD.csv',)
06:46:54   E
06:46:54   E   First differing element 0:
06:46:54   E   's_path\\2023JHUSASchD.csv'
06:46:54   E   's_path/2023JHUSASchD.csv'
06:46:54   E
06:46:54   E   - ('s_path\\2023JHUSASchD.csv',)
06:46:54   E   ?         ^^
06:46:54   E
06:46:54   E   + ('s_path/2023JHUSASchD.csv',)
06:46:54   E   ?         ^
06:46:54   E    : Normalized call args differ for 'is_open':
06:46:54   E     Actual: ('s_path\\2023JHUSASchD.csv',)
06:46:54   E   Expected: ('s_path/2023JHUSASchD.csv',)
06:46:54   _ Test_FileOperations.test_Downloadfilenames_toprocess_ignores_wrong_type_or_extension_filter _
06:46:54   
06:46:54   self = <test.Services.test_fileoperations.Test_FileOperations testMethod=test_Downloadfilenames_toprocess_ignores_wrong_type_or_extension_filter>
06:46:54   
06:46:54       def test_Downloadfilenames_toprocess_ignores_wrong_type_or_extension_filter(self):
06:46:54           server_path, local_path, action, year = 's_path', 'l_path', 'Annual Stmt - Sch D', '2023'
06:46:54           good_file='2023JHUSASchD.csv'
06:46:54           wrong_ext_file='2023JHUSASchD.xlsx'
06:46:54           temp_file='~$2023JHUSASchD_temp.csv'
06:46:54           self.mock_listdir.return_value = [temp_file, wrong_ext_file, good_file]
06:46:54   
06:46:54           with self.assertRaises(FileValidationException):
06:46:54                fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
06:46:54   
06:46:54           wrong_ext_src_path = self._build_path(server_path, wrong_ext_file)
06:46:54           good_file_src_path = self._build_path(server_path, good_file)
06:46:54           good_file_dest_path = self._build_path(local_path, good_file)
06:46:54   
06:46:54           self.assertEqual(self.mock_is_open.call_count, 2, "is_open call count mismatch")
06:46:54           try:
06:46:54                actual_is_open_args = {os.path.normpath(c.args[0]) for c in self.mock_is_open.call_args_list}
06:46:54           except (AttributeError, IndexError, TypeError) as e:
06:46:54                raise AssertionError(f"Could not extract args from mock_is_open.call_args_list: {e}\nList: {self.mock_is_open.call_args_list}")
06:46:54           expected_is_open_args = {os.path.normpath(wrong_ext_src_path), os.path.normpath(good_file_src_path)}
06:46:54   >       self.assertSetEqual(actual_is_open_args, expected_is_open_args, "is_open call arguments mismatch")
06:46:54   E       AssertionError: Items in the first set but not the second:
06:46:54   E       's_path\\2023JHUSASchD.xlsx'
06:46:54   E       's_path\\2023JHUSASchD.csv'
06:46:54   E       Items in the second set but not the first:
06:46:54   E       's_path/2023JHUSASchD.xlsx'
06:46:54   E       's_path/2023JHUSASchD.csv' : is_open call arguments mismatch
06:46:54   
06:46:54   test/Services/test_fileoperations.py:302: AssertionError
06:46:54   ----------------------------- Captured stderr call -----------------------------
06:46:54   Services.dboperations: ERROR    Error in Downloadfilenames_toprocess -Mocked Error Message
06:46:54   root        : ERROR    Exception occurred in Downloadfilenames_toprocess() method :: Mocked Error Message
06:46:54   Traceback (most recent call last):
06:46:54     File "/opt/buildagent/work/9422c6f94111a311/Services/fileoperations.py", line 146, in Downloadfilenames_toprocess
06:46:54       raise CustomException.FileValidationException(errMessageComplete)
06:46:54   Services.CustomException.FileValidationException: Mocked Error Message
06:46:54   ------------------------------ Captured log call -------------------------------
06:46:54   ERROR    Services.dboperations:fileoperations.py:148 Error in Downloadfilenames_toprocess -Mocked Error Message
06:46:54   ERROR    root:fileoperations.py:149 Exception occurred in Downloadfilenames_toprocess() method :: Mocked Error Message
06:46:54   Traceback (most recent call last):
06:46:54     File "/opt/buildagent/work/9422c6f94111a311/Services/fileoperations.py", line 146, in Downloadfilenames_toprocess
06:46:54       raise CustomException.FileValidationException(errMessageComplete)
06:46:54   Services.CustomException.FileValidationException: Mocked Error Message
06:46:54   _____ Test_FileOperations.test_Downloadfilenames_toprocess_qualftc_success _____
06:46:54   
06:46:54   self = <test.Services.test_fileoperations.Test_FileOperations testMethod=test_Downloadfilenames_toprocess_qualftc_success>
06:46:54   
06:46:54       def test_Downloadfilenames_toprocess_qualftc_success(self):
06:46:54           server_path, local_path, action, year = 's_path', 'l_path', 'QualPctFTC', '2023'
06:46:54           filename = '2023QualFTC.xlsx'
06:46:54           self.mock_listdir.return_value = [filename]
06:46:54           result = fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
06:46:54           expected_dest_path = self._build_path(local_path, filename)
06:46:54           expected_src_path = self._build_path(server_path, filename)
06:46:54           self.assertEqual(result, [expected_dest_path])
06:46:54   >       self.assertPathCalledWith(self.mock_is_open, (expected_src_path,))
06:46:54   
06:46:54   test/Services/test_fileoperations.py:177:
06:46:54   _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
06:46:54   test/Services/test_fileoperations.py:96: in assertPathCalledWith
06:46:54       self.assertEqual(normalized_actual, normalized_expected,
06:46:54   E   AssertionError: Tuples differ: ('s_path\\2023QualFTC.xlsx',) != ('s_path/2023QualFTC.xlsx',)
06:46:54   E
06:46:54   E   First differing element 0:
06:46:54   E   's_path\\2023QualFTC.xlsx'
06:46:54   E   's_path/2023QualFTC.xlsx'
06:46:54   E
06:46:54   E   - ('s_path\\2023QualFTC.xlsx',)
06:46:54   E   ?         ^^
06:46:54   E
06:46:54   E   ?         ^
06:46:54   E    : Normalized call args differ for 'is_open':
06:46:54   E     Actual: ('s_path\\2023QualFTC.xlsx',)
06:46:54   E   Expected: ('s_path/2023QualFTC.xlsx',)
06:46:54   _ Test_FileOperations.test_Downloadfilenames_toprocess_wrong_extension_ftcgrossup _
06:46:54   
06:46:54   self = <test.Services.test_fileoperations.Test_FileOperations testMethod=test_Downloadfilenames_toprocess_wrong_extension_ftcgrossup>
06:46:54   
06:46:54       def test_Downloadfilenames_toprocess_wrong_extension_ftcgrossup(self):
06:46:54           server_path, local_path, action, year = 's_path', 'l_path', 'FTCGrossup', '2023'
06:46:54           filename = '2023_Q1_FTCGrossup.csv'
06:46:54           self.mock_listdir.return_value = [filename]
06:46:54           with self.assertRaises(FileValidationException):
06:46:54               fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
06:46:54           expected_src_path = self._build_path(server_path, filename)
06:46:54   >       self.assertPathCalledWith(self.mock_is_open, (expected_src_path,))
06:46:54   
06:46:54   test/Services/test_fileoperations.py:349:
06:46:54   _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
06:46:54   test/Services/test_fileoperations.py:96: in assertPathCalledWith
06:46:54       self.assertEqual(normalized_actual, normalized_expected,
06:46:54   E   AssertionError: Tuples differ: ('s_path\\2023_Q1_FTCGrossup.csv',) != ('s_path/2023_Q1_FTCGrossup.csv',)
06:46:54   E
06:46:54   E   First differing element 0:
06:46:54   E   's_path\\2023_Q1_FTCGrossup.csv'
06:46:54   E   's_path/2023_Q1_FTCGrossup.csv'
06:46:54   E
06:46:54   E   - ('s_path\\2023_Q1_FTCGrossup.csv',)
06:46:54   E   ?         ^^
06:46:54   E
06:46:54   E   + ('s_path/2023_Q1_FTCGrossup.csv',)
06:46:54   E   ?         ^
06:46:54   E    : Normalized call args differ for 'is_open':
06:46:54   E     Actual: ('s_path\\2023_Q1_FTCGrossup.csv',)
06:46:54   E   Expected: ('s_path/2023_Q1_FTCGrossup.csv',)
06:46:54   ----------------------------- Captured stderr call -----------------------------
06:46:54   Services.dboperations: ERROR    Error in Downloadfilenames_toprocess -Mocked Error Message
06:46:54   root        : ERROR    Exception occurred in Downloadfilenames_toprocess() method :: Mocked Error Message
06:46:54   Traceback (most recent call last):
06:46:54     File "/opt/buildagent/work/9422c6f94111a311/Services/fileoperations.py", line 146, in Downloadfilenames_toprocess
06:46:54       raise CustomException.FileValidationException(errMessageComplete)
06:46:54   Services.CustomException.FileValidationException: Mocked Error Message
06:46:54   ------------------------------ Captured log call -------------------------------
06:46:54   ERROR    Services.dboperations:fileoperations.py:148 Error in Downloadfilenames_toprocess -Mocked Error Message
06:46:54   ERROR    root:fileoperations.py:149 Exception occurred in Downloadfilenames_toprocess() method :: Mocked Error Message
06:46:54   Traceback (most recent call last):
06:46:54     File "/opt/buildagent/work/9422c6f94111a311/Services/fileoperations.py", line 146, in Downloadfilenames_toprocess
06:46:54       raise CustomException.FileValidationException(errMessageComplete)
06:46:54   Services.CustomException.FileValidationException: Mocked Error Message
06:46:54   _ Test_FileOperations.test_Downloadfilenames_toprocess_wrong_extension_qualftc _
06:46:54   
06:46:54   self = <test.Services.test_fileoperations.Test_FileOperations testMethod=test_Downloadfilenames_toprocess_wrong_extension_qualftc>
06:46:54   
06:46:54       def test_Downloadfilenames_toprocess_wrong_extension_qualftc(self):
06:46:54           server_path, local_path, action, year = 's_path', 'l_path', 'QualPctFTC', '2023'
06:46:54           filename = '2023QualFTC.csv'
06:46:54           self.mock_listdir.return_value = [filename]
06:46:54           with self.assertRaises(FileValidationException):
06:46:54               fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
06:46:54           expected_src_path = self._build_path(server_path, filename)
06:46:54   >       self.assertPathCalledWith(self.mock_is_open, (expected_src_path,))
06:46:54   
06:46:54   test/Services/test_fileoperations.py:337:
06:46:54   _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
06:46:54   test/Services/test_fileoperations.py:96: in assertPathCalledWith
06:46:54       self.assertEqual(normalized_actual, normalized_expected,
06:46:54   E   AssertionError: Tuples differ: ('s_path\\2023QualFTC.csv',) != ('s_path/2023QualFTC.csv',)
06:46:54   E
06:46:54   E   First differing element 0:
06:46:54   E   's_path\\2023QualFTC.csv'
06:46:54   E   's_path/2023QualFTC.csv'
06:46:54   E
06:46:54   E   - ('s_path\\2023QualFTC.csv',)
06:46:54   E   ?         ^^
06:46:54   E
06:46:54   E   + ('s_path/2023QualFTC.csv',)
06:46:54   E   ?         ^
06:46:54   E    : Normalized call args differ for 'is_open':
06:46:54   E     Actual: ('s_path\\2023QualFTC.csv',)
06:46:54   E   Expected: ('s_path/2023QualFTC.csv',)
06:46:54   ----------------------------- Captured stderr call -----------------------------
06:46:54   Services.dboperations: ERROR    Error in Downloadfilenames_toprocess -Mocked Error Message
06:46:54   root        : ERROR    Exception occurred in Downloadfilenames_toprocess() method :: Mocked Error Message
06:46:54   Traceback (most recent call last):
06:46:54     File "/opt/buildagent/work/9422c6f94111a311/Services/fileoperations.py", line 146, in Downloadfilenames_toprocess
06:46:54       raise CustomException.FileValidationException(errMessageComplete)
06:46:54   Services.CustomException.FileValidationException: Mocked Error Message
06:46:54   ------------------------------ Captured log call -------------------------------
06:46:54   ERROR    Services.dboperations:fileoperations.py:148 Error in Downloadfilenames_toprocess -Mocked Error Message
06:46:54   ERROR    root:fileoperations.py:149 Exception occurred in Downloadfilenames_toprocess() method :: Mocked Error Message
06:46:54   Traceback (most recent call last):
06:46:54     File "/opt/buildagent/work/9422c6f94111a311/Services/fileoperations.py", line 146, in Downloadfilenames_toprocess
06:46:54       raise CustomException.FileValidationException(errMessageComplete)
06:46:54   Services.CustomException.FileValidationException: Mocked Error Message
06:46:54   __ Test_FileOperations.test_Downloadfilenames_toprocess_wrong_file_type_schd ___
06:46:54   
06:46:54   self = <test.Services.test_fileoperations.Test_FileOperations testMethod=test_Downloadfilenames_toprocess_wrong_file_type_schd>
06:46:54   
06:46:54       def test_Downloadfilenames_toprocess_wrong_file_type_schd(self):
06:46:54           server_path, local_path, action, year = 's_path', 'l_path', 'Annual Stmt - Sch D', '2023'
06:46:54           filename = '2023JHUSASchD.xlsx'
06:46:54           self.mock_listdir.return_value = [filename]
06:46:54           with self.assertRaises(FileValidationException):
06:46:54               fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
06:46:54           expected_src_path = self._build_path(server_path, filename)
06:46:54   >       self.assertPathCalledWith(self.mock_is_open, (expected_src_path,))
06:46:54   
06:46:54   test/Services/test_fileoperations.py:218:
06:46:54   _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
06:46:54   test/Services/test_fileoperations.py:96: in assertPathCalledWith
06:46:54       self.assertEqual(normalized_actual, normalized_expected,
06:46:54   E   AssertionError: Tuples differ: ('s_path\\2023JHUSASchD.xlsx',) != ('s_path/2023JHUSASchD.xlsx',)
06:46:54   E
06:46:54   E   First differing element 0:
06:46:54   E   's_path\\2023JHUSASchD.xlsx'
06:46:54   E   's_path/2023JHUSASchD.xlsx'
06:46:54   E
06:46:54   E   - ('s_path\\2023JHUSASchD.xlsx',)
06:46:54   E   ?         ^^
06:46:54   E
06:46:54   E   + ('s_path/2023JHUSASchD.xlsx',)
06:46:54   E   ?         ^
06:46:54   E    : Normalized call args differ for 'is_open':
06:46:54   E     Actual: ('s_path\\2023JHUSASchD.xlsx',)
06:46:54   E   Expected: ('s_path/2023JHUSASchD.xlsx',)
06:46:54   ----------------------------- Captured stderr call -----------------------------
06:46:54   Services.dboperations: ERROR    Error in Downloadfilenames_toprocess -Mocked Error Message
06:46:54   root        : ERROR    Exception occurred in Downloadfilenames_toprocess() method :: Mocked Error Message
06:46:54   Traceback (most recent call last):
06:46:54     File "/opt/buildagent/work/9422c6f94111a311/Services/fileoperations.py", line 146, in Downloadfilenames_toprocess
06:46:54       raise CustomException.FileValidationException(errMessageComplete)
06:46:54   Services.CustomException.FileValidationException: Mocked Error Message
06:46:54   ------------------------------ Captured log call -------------------------------
06:46:54   ERROR    Services.dboperations:fileoperations.py:148 Error in Downloadfilenames_toprocess -Mocked Error Message
06:46:54   ERROR    root:fileoperations.py:149 Exception occurred in Downloadfilenames_toprocess() method :: Mocked Error Message
06:46:54   Traceback (most recent call last):
06:46:54     File "/opt/buildagent/work/9422c6f94111a311/Services/fileoperations.py", line 146, in Downloadfilenames_toprocess
06:46:54       raise CustomException.FileValidationException(errMessageComplete)
06:46:54   Services.CustomException.FileValidationException: Mocked Error Message
06:46:54   =============================== warnings summary ===============================
06:46:54   ../../../../usr/local/lib/python3.9/dist-packages/pandas/compat/numpy/__init__.py:10
06:46:54     /usr/local/lib/python3.9/dist-packages/pandas/compat/numpy/__init__.py:10: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
06:46:54       _nlv = LooseVersion(_np_version)
06:46:54   
06:46:54   ../../../../usr/local/lib/python3.9/dist-packages/pandas/compat/numpy/__init__.py:11
06:46:54     /usr/local/lib/python3.9/dist-packages/pandas/compat/numpy/__init__.py:11: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
06:46:54       np_version_under1p17 = _nlv < LooseVersion("1.17")
06:46:54   
06:46:54   ../../../../usr/local/lib/python3.9/dist-packages/pandas/compat/numpy/__init__.py:12
06:46:54     /usr/local/lib/python3.9/dist-packages/pandas/compat/numpy/__init__.py:12: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
06:46:54       np_version_under1p18 = _nlv < LooseVersion("1.18")
06:46:54   
06:46:54   ../../../../usr/local/lib/python3.9/dist-packages/pandas/compat/numpy/__init__.py:13
06:46:54     /usr/local/lib/python3.9/dist-packages/pandas/compat/numpy/__init__.py:13: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
06:46:54       _np_version_under1p19 = _nlv < LooseVersion("1.19")
06:46:54   
06:46:54   ../../../../usr/local/lib/python3.9/dist-packages/pandas/compat/numpy/__init__.py:14
06:46:54     /usr/local/lib/python3.9/dist-packages/pandas/compat/numpy/__init__.py:14: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
06:46:54       _np_version_under1p20 = _nlv < LooseVersion("1.20")
06:46:54   
06:46:54   ../../../../usr/local/lib/python3.9/dist-packages/setuptools/_distutils/version.py:337
06:46:54     /usr/local/lib/python3.9/dist-packages/setuptools/_distutils/version.py:337: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
06:46:54       other = LooseVersion(other)
06:46:54   
06:46:54   ../../../../usr/local/lib/python3.9/dist-packages/pandas/compat/numpy/function.py:120
06:46:54   ../../../../usr/local/lib/python3.9/dist-packages/pandas/compat/numpy/function.py:120
06:46:54     /usr/local/lib/python3.9/dist-packages/pandas/compat/numpy/function.py:120: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
06:46:54       if LooseVersion(__version__) >= LooseVersion("1.17.0"):
06:46:54   
06:46:54   ../../../../usr/local/lib/python3.9/dist-packages/flask_sqlalchemy/__init__.py:14
06:46:54   ../../../../usr/local/lib/python3.9/dist-packages/flask_sqlalchemy/__init__.py:14
06:46:54     /usr/local/lib/python3.9/dist-packages/flask_sqlalchemy/__init__.py:14: DeprecationWarning: '_app_ctx_stack' is deprecated and will be removed in Flask 2.3.
06:46:54       from flask import _app_ctx_stack, abort, current_app, request
06:46:54   
06:46:54   -- Docs: https://docs.pytest.org/en/stable/how-to/capture-warnings.html
06:46:54   
06:46:54   ----------- coverage: platform linux, python 3.9.5-final-0 -----------
06:46:54   Coverage XML written to file coverage.xml
06:46:54   
06:46:54   =========================== short test summary info ============================
06:46:54   FAILED test/Services/test_fileoperations.py::Test_FileOperations::test_Downloadfilenames_toprocess_annual_schd_success - AssertionError: Tuples differ: ('s_path\\2023JHUSASchD.csv',) != ('s_path/2023JHUSASchD.csv',)
06:46:54   
06:46:54   First differing element 0:
06:46:54   's_path\\2023JHUSASchD.csv'
06:46:54   's_path/2023JHUSASchD.csv'
06:46:54   
06:46:54   - ('s_path\\2023JHUSASchD.csv',)
06:46:54   ?         ^^
06:46:54   
06:46:54   + ('s_path/2023JHUSASchD.csv',)
06:46:54   ?         ^
06:46:54    : Normalized call args differ for 'is_open':
06:46:54     Actual: ('s_path\\2023JHUSASchD.csv',)
06:46:54   Expected: ('s_path/2023JHUSASchD.csv',)
06:46:54   FAILED test/Services/test_fileoperations.py::Test_FileOperations::test_Downloadfilenames_toprocess_dest_dir_does_not_exist - AssertionError: Tuples differ: ('s_path\\2023JHUSASchD.csv',) != ('s_path/2023JHUSASchD.csv',)
06:46:54   
06:46:54   First differing element 0:
06:46:54   's_path\\2023JHUSASchD.csv'
06:46:54   's_path/2023JHUSASchD.csv'
06:46:54   
06:46:54   - ('s_path\\2023JHUSASchD.csv',)
06:46:54   ?         ^^
06:46:54   
06:46:54   + ('s_path/2023JHUSASchD.csv',)
06:46:54   ?         ^
06:46:54    : Normalized call args differ for 'is_open':
06:46:54     Actual: ('s_path\\2023JHUSASchD.csv',)
06:46:54   Expected: ('s_path/2023JHUSASchD.csv',)
06:46:54   FAILED test/Services/test_fileoperations.py::Test_FileOperations::test_Downloadfilenames_toprocess_file_open - AssertionError: Tuples differ: ('s_path\\2023JHUSASchD.csv',) != ('s_path/2023JHUSASchD.csv',)
06:46:54   
06:46:54   First differing element 0:
06:46:54   's_path\\2023JHUSASchD.csv'
06:46:54   's_path/2023JHUSASchD.csv'
06:46:54   
06:46:54   - ('s_path\\2023JHUSASchD.csv',)
06:46:54   ?         ^^
06:46:54   
06:46:54   + ('s_path/2023JHUSASchD.csv',)
06:46:54   ?         ^
06:46:54    : Normalized call args differ for 'is_open':
06:46:54     Actual: ('s_path\\2023JHUSASchD.csv',)
06:46:54   Expected: ('s_path/2023JHUSASchD.csv',)
06:46:54   FAILED test/Services/test_fileoperations.py::Test_FileOperations::test_Downloadfilenames_toprocess_ftcgrossup_success - AssertionError: Items in the first set but not the second:
06:46:54   's_path\\2023_Q1_FTCGrossup.xlsx'
06:46:54   's_path\\2023_Q2_FTCGrossup.xlsx'
06:46:54   's_path\\2023_Q3_FTCGrossup.xlsx'
06:46:54   's_path\\2023_Q4_FTCGrossup.xlsx'
06:46:54   Items in the second set but not the first:
06:46:54   's_path/2023_Q1_FTCGrossup.xlsx'
06:46:54   's_path/2023_Q4_FTCGrossup.xlsx'
06:46:54   's_path/2023_Q2_FTCGrossup.xlsx'
06:46:54   's_path/2023_Q3_FTCGrossup.xlsx' : is_open call arguments mismatch
06:46:54   FAILED test/Services/test_fileoperations.py::Test_FileOperations::test_Downloadfilenames_toprocess_ignores_temp_files - AssertionError: Tuples differ: ('s_path\\2023JHUSASchD.csv',) != ('s_path/2023JHUSASchD.csv',)
06:46:54   
06:46:54   First differing element 0:
06:46:54   's_path\\2023JHUSASchD.csv'
06:46:54   's_path/2023JHUSASchD.csv'
06:46:54   
06:46:54   - ('s_path\\2023JHUSASchD.csv',)
06:46:54   ?         ^^
06:46:54   
06:46:54   + ('s_path/2023JHUSASchD.csv',)
06:46:54   ?         ^
06:46:54    : Normalized call args differ for 'is_open':
06:46:54     Actual: ('s_path\\2023JHUSASchD.csv',)
06:46:54   Expected: ('s_path/2023JHUSASchD.csv',)
06:46:54   FAILED test/Services/test_fileoperations.py::Test_FileOperations::test_Downloadfilenames_toprocess_ignores_wrong_type_or_extension_filter - AssertionError: Items in the first set but not the second:
06:46:54   's_path\\2023JHUSASchD.xlsx'
06:46:54   's_path\\2023JHUSASchD.csv'
06:46:54   Items in the second set but not the first:
06:46:54   's_path/2023JHUSASchD.xlsx'
06:46:54   's_path/2023JHUSASchD.csv' : is_open call arguments mismatch
06:46:54   FAILED test/Services/test_fileoperations.py::Test_FileOperations::test_Downloadfilenames_toprocess_qualftc_success - AssertionError: Tuples differ: ('s_path\\2023QualFTC.xlsx',) != ('s_path/2023QualFTC.xlsx',)
06:46:54   
06:46:54   First differing element 0:
06:46:54   's_path\\2023QualFTC.xlsx'
06:46:54   's_path/2023QualFTC.xlsx'
06:46:54   
06:46:54   - ('s_path\\2023QualFTC.xlsx',)
06:46:54   ?         ^^
06:46:54   
06:46:54   + ('s_path/2023QualFTC.xlsx',)
06:46:54   ?         ^
06:46:54    : Normalized call args differ for 'is_open':
06:46:54     Actual: ('s_path\\2023QualFTC.xlsx',)
06:46:54   Expected: ('s_path/2023QualFTC.xlsx',)
06:46:54   FAILED test/Services/test_fileoperations.py::Test_FileOperations::test_Downloadfilenames_toprocess_wrong_extension_ftcgrossup - AssertionError: Tuples differ: ('s_path\\2023_Q1_FTCGrossup.csv',) != ('s_path/2023_Q1_FTCGrossup.csv',)
06:46:54   
06:46:54   First differing element 0:
06:46:54   's_path\\2023_Q1_FTCGrossup.csv'
06:46:54   's_path/2023_Q1_FTCGrossup.csv'
06:46:54   
06:46:54   - ('s_path\\2023_Q1_FTCGrossup.csv',)
06:46:54   ?         ^^
06:46:54   
06:46:54   + ('s_path/2023_Q1_FTCGrossup.csv',)
06:46:54   ?         ^
06:46:54    : Normalized call args differ for 'is_open':
06:46:54     Actual: ('s_path\\2023_Q1_FTCGrossup.csv',)
06:46:54   Expected: ('s_path/2023_Q1_FTCGrossup.csv',)
06:46:54   FAILED test/Services/test_fileoperations.py::Test_FileOperations::test_Downloadfilenames_toprocess_wrong_extension_qualftc - AssertionError: Tuples differ: ('s_path\\2023QualFTC.csv',) != ('s_path/2023QualFTC.csv',)
06:46:54   
06:46:54   First differing element 0:
06:46:54   's_path\\2023QualFTC.csv'
06:46:54   's_path/2023QualFTC.csv'
06:46:54   
06:46:54   - ('s_path\\2023QualFTC.csv',)
06:46:54   ?         ^^
06:46:54   
06:46:54   + ('s_path/2023QualFTC.csv',)
06:46:54   ?         ^
06:46:54    : Normalized call args differ for 'is_open':
06:46:54     Actual: ('s_path\\2023QualFTC.csv',)
06:46:54   Expected: ('s_path/2023QualFTC.csv',)
06:46:54   FAILED test/Services/test_fileoperations.py::Test_FileOperations::test_Downloadfilenames_toprocess_wrong_file_type_schd - AssertionError: Tuples differ: ('s_path\\2023JHUSASchD.xlsx',) != ('s_path/2023JHUSASchD.xlsx',)
06:46:54   
06:46:54   First differing element 0:
06:46:54   's_path\\2023JHUSASchD.xlsx'
06:46:54   's_path/2023JHUSASchD.xlsx'
06:46:54   
06:46:54   - ('s_path\\2023JHUSASchD.xlsx',)
06:46:54    : Normalized call args differ for 'is_open':
06:46:54     Actual: ('s_path\\2023JHUSASchD.xlsx',)
06:46:54   Expected: ('s_path/2023JHUSASchD.xlsx',)
06:46:54   ================== 10 failed, 79 passed, 10 warnings in 3.14s ==================
06:46:54   Process exited with code 1
06:46:54   Process exited with code 1 (Step: PYTHON PYTEST : LINUX (Command Line))
06:46:54   Step PYTHON PYTEST : LINUX (Command Line) failed
06:46:54 Step 8/16: PYTHON PYTEST : WINDOWS (PowerShell)
06:46:54 Step 9/16: SONARQUBE (SonarQube Runner)
06:46:54 Step 10/16: SONAR QUALITY GATE CHECK (PowerShell)
06:46:54 Step 11/16: SONAR QUALITY GATE CHECK : LINUX (PowerShell)
06:46:54 Step 12/16: PYTHON AND PIP VERSION (Command Line)
06:46:54 Step 13/16: PIP INSTALL (Command Line)
06:46:54 Step 14/16: CREATE LOGS FOLDER (PowerShell)
06:46:54 Step 15/16: PYTHON PYTEST (Command Line)
06:46:54 Step 16/16: SONAR QUALITY GATE CHECK (PowerShell)
06:46:54 Publishing internal artifacts
06:46:55 Build is failed. Artifacts will not be published for this build
06:46:55 Build finished
