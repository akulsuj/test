import unittest
from unittest.mock import MagicMock, patch, call
import os
import datetime
import shutil

# --- Imports ---
import Services.fileoperations as fo
from Services.CustomException import FileValidationException
import globalvars as gvar

class Test_FileOperations(unittest.TestCase):

    def setUp(self):
        """Set up test fixtures, mocks before each test method."""
        # 1. Patch the module-level dbops_obj INSTANCE directly
        self.patcher_fo_dbops_obj = patch('Services.fileoperations.dbops_obj', spec=True)
        self.mock_dbops_obj = self.patcher_fo_dbops_obj.start()

        # 2. Configure the instance mock
        self.mock_dbops_obj.BuildErrorMessage.return_value = "Mocked Error Message"
        mock_settings = [
            MagicMock(settingName="Valid_Company", settingValue="JHUSA"), MagicMock(settingName="Valid_Company", settingValue="JHIL"),
            MagicMock(settingName="Valid_Quarter", settingValue="Q1"), MagicMock(settingName="Valid_Quarter", settingValue="Q2"),
            MagicMock(settingName="Valid_Quarter", settingValue="Q3"), MagicMock(settingName="Valid_Quarter", settingValue="Q4"),
            MagicMock(settingName="Filename_AnnStmtSchD", settingValue="SchD"), MagicMock(settingName="Filename_QualFTC", settingValue="QualFTC"),
            MagicMock(settingName="Filename_FTCGrossup", settingValue="FTCGrossup"),
        ]
        self.mock_dbops_obj.SadrdSysSettings.return_value = mock_settings
        self.mock_dbops_obj.SADRD_Sys_Message.return_value = MagicMock()

        # 3. Assign the settings to gvar
        gvar.sadrd_settings = self.mock_dbops_obj.SadrdSysSettings()
        gvar.sadrd_ErrMessages = self.mock_dbops_obj.SADRD_Sys_Message()
        gvar.user_id = 'test_user_123'

        # --- Patch OS, shutil, and is_open functions ---
        self.patcher_exists = patch('os.path.exists', return_value=True)
        self.patcher_makedirs = patch('os.makedirs')
        self.patcher_listdir = patch('os.listdir')
        self.patcher_rename = patch('os.rename', return_value=None)
        self.patcher_copyfile = patch('shutil.copyfile')
        self.patcher_os_path_join = patch('os.path.join', side_effect=os.path.join)
        self.patcher_is_open = patch('Services.fileoperations.is_open', return_value=False)

        self.mock_exists = self.patcher_exists.start()
        self.mock_makedirs = self.patcher_makedirs.start()
        self.mock_listdir = self.patcher_listdir.start()
        self.mock_rename = self.patcher_rename.start()
        self.mock_copyfile = self.patcher_copyfile.start()
        self.mock_os_path_join = self.patcher_os_path_join.start()
        self.mock_is_open = self.patcher_is_open.start()

    def tearDown(self):
        """Tear down test fixtures, stop mocks after each test method."""
        patch.stopall()

    # --- Helper ---
    def _build_path(self, *args):
        return self.mock_os_path_join(*args)

    # ==================================================================
    # --- Test Methods (Corrected) ---
    # ==================================================================

    def test_getinpfilenames_toprocess(self):
        folderpath = 'test_folder'; inpLoadFolder = 'input_files'
        full_dir_path = self._build_path(folderpath, inpLoadFolder)
        self.mock_listdir.return_value = ['file1.xlsx', 'file2.xlsm', 'file3.csv', 'file4.txt']
        expected_files = [ self._build_path(full_dir_path, f) for f in ['file1.xlsx', 'file2.xlsm', 'file3.csv']]
        result = fo.getinpfilenames_toprocess(folderpath, inpLoadFolder)
        self.mock_os_path_join.assert_any_call(folderpath, inpLoadFolder)
        self.mock_os_path_join.assert_any_call(full_dir_path, 'file1.xlsx')
        self.assertEqual(sorted(result), sorted(expected_files))
        self.assertNotIn(self._build_path(full_dir_path, 'file4.txt'), result)
        self.mock_listdir.assert_called_once_with(full_dir_path)

    def test_Downloadfilenames_toprocess_annual_schd_success(self):
        server_path, local_path, action, year = 's', 'l', 'Annual Stmt - Sch D', '2023'
        filename = '2023JHUSASchD.csv'
        self.mock_listdir.return_value = [filename]
        result = fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
        expected_dest_os_join = self._build_path(local_path, filename)
        expected_src_os_join = self._build_path(server_path, filename)
        expected_dest_copyfile = local_path + '//' + filename
        expected_src_copyfile = server_path + '//' + filename
        self.assertEqual(result, [expected_dest_os_join])
        self.mock_is_open.assert_called_once_with(expected_src_os_join)
        self.mock_copyfile.assert_called_once_with(expected_src_copyfile, expected_dest_copyfile)
        self.mock_dbops_obj.insert_actionLog.assert_called_once()

    def test_Downloadfilenames_toprocess_qualftc_success(self):
        server_path, local_path, action, year = 's', 'l', 'QualPctFTC', '2023'
        filename = '2023QualFTC.xlsx'
        self.mock_listdir.return_value = [filename]
        result = fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
        expected_dest_os_join = self._build_path(local_path, filename)
        expected_src_os_join = self._build_path(server_path, filename)
        expected_dest_copyfile = local_path + '//' + filename
        expected_src_copyfile = server_path + '//' + filename
        self.assertEqual(result, [expected_dest_os_join])
        self.mock_is_open.assert_called_once_with(expected_src_os_join)
        self.mock_copyfile.assert_called_once_with(expected_src_copyfile, expected_dest_copyfile)
        self.mock_dbops_obj.insert_actionLog.assert_called_once()

    def test_Downloadfilenames_toprocess_ftcgrossup_success(self):
        server_path, local_path, action, year = 's', 'l', 'FTCGrossup', '2023'
        files = [f'2023_Q{i}_FTCGrossup.xlsx' for i in range(1, 5)]
        self.mock_listdir.return_value = files
        result = fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
        expected_dest_files_os_join = [self._build_path(local_path, f) for f in files]
        expected_src_files_os_join = [self._build_path(server_path, f) for f in files]
        expected_is_open_calls = [call(src) for src in expected_src_files_os_join]
        self.assertEqual(sorted(result), sorted(expected_dest_files_os_join))
        self.mock_is_open.assert_has_calls(expected_is_open_calls, any_order=True)
        self.assertEqual(self.mock_copyfile.call_count, 4)
        self.mock_copyfile.assert_any_call(server_path + '//' + files[0], local_path + '//' + files[0])
        self.mock_dbops_obj.insert_actionLog.assert_called_once()

    def test_Downloadfilenames_toprocess_wrong_file_type_schd(self):
        server_path, local_path, action, year = 's', 'l', 'Annual Stmt - Sch D', '2023'
        filename = '2023JHUSASchD.xlsx'
        self.mock_listdir.return_value = [filename]
        with self.assertRaises(FileValidationException):
            fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
        expected_src_os_join = self._build_path(server_path, filename)
        self.mock_is_open.assert_called_once_with(expected_src_os_join)
        self.mock_dbops_obj.BuildErrorMessage.assert_called_once_with(f'E003,{filename},None; E002,,')
        self.mock_copyfile.assert_not_called()
        self.mock_dbops_obj.insert_actionLog.assert_called_once()

    def test_Downloadfilenames_toprocess_file_open(self):
        server_path, local_path, action, year = 's', 'l', 'Annual Stmt - Sch D', '2023'
        filename = '2023JHUSASchD.csv'
        self.mock_listdir.return_value = [filename]; self.mock_is_open.return_value = True
        with self.assertRaises(FileValidationException):
            fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
        expected_src_os_join = self._build_path(server_path, filename)
        # self.mock_is_open.assert_called_once_with(expected_src_os_join) # is_open mock check
        self.mock_dbops_obj.BuildErrorMessage.assert_called_once_with(f'E006,{filename},None; E002,,')
        self.mock_copyfile.assert_not_called()
        self.mock_dbops_obj.insert_actionLog.assert_called_once()

    @patch('Services.fileoperations.getinpfilenames_toprocess')
    def test_ExtractFilesToLoad(self, mock_getinp):
        folderpath, action = 'local_folder', 'SomeAction'
        expected_list = [os.path.join('lf','f1.csv'), os.path.join('lf','f2.xlsx')]
        mock_getinp.return_value = expected_list
        result = fo.ExtractFilesToLoad(folderpath, action)
        mock_getinp.assert_called_once_with(folderpath, "")
        self.assertEqual(result, expected_list)

    @patch('Services.fileoperations.Downloadfilenames_toprocess')
    def test_DownloadServerFilesToLoad(self, mock_download):
        server_path, local_path, action, year = 's', 'l', 'ActionX', '2024'
        expected_list = [os.path.join(local_path, 'file1.csv')]
        mock_download.return_value = expected_list
        result = fo.DownloadServerFilesToLoad(server_path, local_path, action, year)
        mock_download.assert_called_once_with(server_path, local_path, action, year)
        self.assertEqual(result, expected_list)

    def test_Downloadfilenames_toprocess_dest_dir_does_not_exist(self):
        self.mock_exists.return_value = False
        self.mock_listdir.return_value = ['2023JHUSASchD.csv']
        local_path = 'local_files'
        fo.Downloadfilenames_toprocess('server', local_path, 'Annual Stmt - Sch D', '2023')
        self.mock_exists.assert_called_once_with(local_path)
        self.mock_makedirs.assert_called_once_with(local_path)

    def test_Downloadfilenames_toprocess_ignores_temp_files(self):
        server_path, local_path, action, year = 's', 'l', 'Annual Stmt - Sch D', '2023'
        good_file, temp_file = '2023JHUSASchD.csv', '~$2023JHUSASchD.csv'
        self.mock_listdir.return_value = [temp_file, good_file]
        result = fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
        expected_dest_os_join = self._build_path(local_path, good_file)
        expected_src_os_join = self._build_path(server_path, good_file)
        expected_dest_copyfile = local_path + '//' + good_file
        expected_src_copyfile = server_path + '//' + good_file
        self.assertEqual(result, [expected_dest_os_join])
        self.mock_copyfile.assert_called_once_with(expected_src_copyfile, expected_dest_copyfile)
        self.mock_is_open.assert_called_once_with(expected_src_os_join)
        self.mock_dbops_obj.insert_actionLog.assert_called_once()

    def test_Downloadfilenames_toprocess_ignores_wrong_type_or_extension_filter(self):
        server_path, local_path, action, year = 's', 'l', 'Annual Stmt - Sch D', '2023'
        good_file='2023JHUSASchD.csv'; wrong_type='2023Wrong.csv'; wrong_ext='2023SchD.txt'
        self.mock_listdir.return_value = [wrong_type, wrong_ext, good_file]
        result = fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
        expected_dest_os_join = self._build_path(local_path, good_file)
        expected_src_os_join = self._build_path(server_path, good_file)
        expected_dest_copyfile = local_path + '//' + good_file
        expected_src_copyfile = server_path + '//' + good_file
        self.assertEqual(result, [expected_dest_os_join])
        self.mock_copyfile.assert_called_once_with(expected_src_copyfile, expected_dest_copyfile)
        self.mock_is_open.assert_called_once_with(expected_src_os_join)
        self.mock_dbops_obj.insert_actionLog.assert_called_once()

    def test_Downloadfilenames_toprocess_empty_after_initial_filter(self):
        server_path, local_path, action, year = 's', 'l', 'Annual Stmt - Sch D', '2023'
        self.mock_listdir.return_value = ['~$f.csv', 'f.txt', 'Wrong.xlsx']
        with self.assertRaises(FileValidationException):
            fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
        self.mock_dbops_obj.BuildErrorMessage.assert_called_once_with('E002,,')
        self.mock_copyfile.assert_not_called()
        self.mock_is_open.assert_not_called()
        self.mock_dbops_obj.insert_actionLog.assert_called_once()

    def test_Downloadfilenames_toprocess_wrong_extension_qualftc(self):
        server_path, local_path, action, year = 's', 'l', 'QualPctFTC', '2023'
        filename = '2023QualFTC.csv'
        self.mock_listdir.return_value = [filename]
        with self.assertRaises(FileValidationException):
            fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
        expected_src_os_join = self._build_path(server_path, filename)
        self.mock_is_open.assert_called_once_with(expected_src_os_join)
        self.mock_dbops_obj.BuildErrorMessage.assert_called_once_with(f'E003,{filename},None; E002,,')
        self.mock_copyfile.assert_not_called()
        self.mock_dbops_obj.insert_actionLog.assert_called_once()

    def test_Downloadfilenames_toprocess_wrong_extension_ftcgrossup(self):
        server_path, local_path, action, year = 's', 'l', 'FTCGrossup', '2023'
        filename = '2023_Q1_FTCGrossup.csv'
        self.mock_listdir.return_value = [filename]
        with self.assertRaises(FileValidationException):
            fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
        expected_src_os_join = self._build_path(server_path, filename)
        self.mock_is_open.assert_called_once_with(expected_src_os_join)
        self.mock_dbops_obj.BuildErrorMessage.assert_called_once_with(f'E003,{filename},None; E002,,')
        self.mock_copyfile.assert_not_called()
        self.mock_dbops_obj.insert_actionLog.assert_called_once()

    def test_Downloadfilenames_toprocess_invalid_quarter_ftcgrossup(self):
        server_path, local_path, action, year = 's', 'l', 'FTCGrossup', '2023'
        filename = '2023_Q5_FTCGrossup.xlsx'
        self.mock_listdir.return_value = [filename]
        with self.assertRaises(FileValidationException):
            fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
        expected_src_os_join = self._build_path(server_path, filename)
        self.mock_is_open.assert_called_once_with(expected_src_os_join)
        self.mock_dbops_obj.BuildErrorMessage.assert_called_once_with(f'E013,{filename},None; E002,,')
        self.mock_copyfile.assert_not_called()
        self.mock_dbops_obj.insert_actionLog.assert_called_once()

    def test_Downloadfilenames_toprocess_invalid_company_schd(self):
        server_path, local_path, action, year = 's', 'l', 'Annual Stmt - Sch D', '2023'
        filename = '2023XYZSchD.csv'
        self.mock_listdir.return_value = [filename]
        with self.assertRaises(FileValidationException):
            fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
        expected_src_os_join = self._build_path(server_path, filename)
        self.mock_is_open.assert_called_once_with(expected_src_os_join)
        self.mock_dbops_obj.BuildErrorMessage.assert_called_once_with(f'E011,{filename},None; E002,,')
        self.mock_copyfile.assert_not_called()
        self.mock_dbops_obj.insert_actionLog.assert_called_once()

    def test_Downloadfilenames_toprocess_wrong_year(self):
        server_path, local_path, action, input_year = 's', 'l', 'Annual Stmt - Sch D', '2023'
        filename = '2022JHUSASchD.csv'
        self.mock_listdir.return_value = [filename]
        with self.assertRaises(FileValidationException):
            fo.Downloadfilenames_toprocess(server_path, local_path, action, input_year)
        expected_src_os_join = self._build_path(server_path, filename)
        self.mock_is_open.assert_called_once_with(expected_src_os_join)
        self.mock_dbops_obj.BuildErrorMessage.assert_called_once_with(f'E012,{filename},None; E002,,')
        self.mock_copyfile.assert_not_called()
        self.mock_dbops_obj.insert_actionLog.assert_called_once()

    def test_Downloadfilenames_toprocess_unexpected_exception_during_copy(self):
        server_path, local_path, action, year = 's', 'l', 'Annual Stmt - Sch D', '2023'
        filename = '2023JHUSASchD.csv'
        self.mock_listdir.return_value = [filename]
        self.mock_is_open.return_value = False
        self.mock_copyfile.side_effect = OSError("Disk full")
        with self.assertRaises(OSError) as cm:
            fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
        self.assertEqual(str(cm.exception), "Disk full")
        expected_src_os_join = self._build_path(server_path, filename)
        expected_dest_copyfile = local_path + '//' + filename
        expected_src_copyfile = server_path + '//' + filename
        self.mock_is_open.assert_called_once_with(expected_src_os_join)
        self.mock_copyfile.assert_called_once_with(expected_src_copyfile, expected_dest_copyfile)
        # CORRECTED: Log is NOT called if exception jumps execution to except block before log line
        self.mock_dbops_obj.insert_actionLog.assert_not_called()

    def test_Downloadfilenames_toprocess_no_files_in_server_dir(self):
        server_path, local_path, action, year = 's', 'l', 'Annual Stmt - Sch D', '2023'
        self.mock_listdir.return_value = []
        with self.assertRaises(FileValidationException):
            fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
        self.mock_dbops_obj.BuildErrorMessage.assert_called_once_with('E002,,')
        self.mock_copyfile.assert_not_called()
        self.mock_is_open.assert_not_called()
        self.mock_dbops_obj.insert_actionLog.assert_called_once()

    def test_Downloadfilenames_toprocess_missing_quarters_ftcgrossup(self):
        server_path, local_path, action, year = 's', 'l', 'FTCGrossup', '2023'
        files_present = [ '2023_Q1_FTCGrossup.xlsx', '2023_Q3_FTCGrossup.xlsx']
        self.mock_listdir.return_value = files_present
        expected_src_files_os_join = [self._build_path(server_path, f) for f in files_present]
        expected_is_open_calls = [call(src) for src in expected_src_files_os_join]
        with self.assertRaises(FileValidationException):
            fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
        self.assertEqual(self.mock_copyfile.call_count, 2)
        self.mock_is_open.assert_has_calls(expected_is_open_calls, any_order=True)
        # CORRECTED: Check E028 raised *after* try block
        self.mock_dbops_obj.BuildErrorMessage.assert_called_once_with('E028,,')
        self.mock_dbops_obj.insert_actionLog.assert_called_once()

    def test_Downloadfilenames_toprocess_not_exactly_4_files_ftcgrossup(self):
        server_path, local_path, action, year = 's', 'l', 'FTCGrossup', '2023'
        files_present = [ '2023_Q1_FTCGrossup.xlsx', '2023_Q2_FTCGrossup.xlsx', '2023_Q3_FTCGrossup.xlsx']
        self.mock_listdir.return_value = files_present
        expected_src_files_os_join = [self._build_path(server_path, f) for f in files_present]
        expected_is_open_calls = [call(src) for src in expected_src_files_os_join]
        with self.assertRaises(FileValidationException):
            fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
        self.assertEqual(self.mock_copyfile.call_count, 3)
        self.mock_is_open.assert_has_calls(expected_is_open_calls, any_order=True)
        self.mock_dbops_obj.BuildErrorMessage.assert_called_once_with('E028,,')
        self.mock_dbops_obj.insert_actionLog.assert_called_once()

    # --- Tests for actual is_open helper function ---

    @patch('os.path.exists')
    @patch('os.rename')
    def test_actual_is_open_success_not_open(self, mock_rename_local, mock_exists_local):
        filepath = "file.txt"
        mock_exists_local.return_value = True
        mock_rename_local.return_value = None
        patcher_to_manage = self.patcher_is_open
        try: patcher_to_manage.stop()
        finally: # Ensure is_open mock is stopped before calling real function
            result = fo.is_open(filepath)
            patcher_to_manage.start() # Restart class mock
        self.assertFalse(result)
        mock_exists_local.assert_called_once_with(filepath)
        mock_rename_local.assert_called_once_with(filepath, filepath)

    @patch('os.path.exists')
    @patch('os.rename')
    def test_actual_is_open_fails_permission_error_is_open(self, mock_rename_local, mock_exists_local):
        filepath = "file.txt"
        mock_exists_local.return_value = True
        mock_rename_local.side_effect = PermissionError
        patcher_to_manage = self.patcher_is_open
        try: patcher_to_manage.stop()
        finally:
            result = fo.is_open(filepath)
            patcher_to_manage.start()
        self.assertTrue(result)
        mock_exists_local.assert_called_once_with(filepath)
        mock_rename_local.assert_called_once_with(filepath, filepath)

    @patch('os.path.exists')
    @patch('os.rename')
    def test_actual_is_open_file_not_exist_raises_error(self, mock_rename_local, mock_exists_local):
        filepath = "non_existent_file.txt"
        mock_exists_local.return_value = False
        patcher_to_manage = self.patcher_is_open
        try: patcher_to_manage.stop()
        finally:
            with self.assertRaises(NameError):
                 fo.is_open(filepath)
            patcher_to_manage.start()
        mock_exists_local.assert_called_once_with(filepath)
        mock_rename_local.assert_not_called()

# --- Standard test runner ---
if __name__ == '__main__':
    unittest.main(argv=['first-arg-is-ignored'], exit=False)



08:24:33   Coverage XML written to file coverage.xml
08:24:33   
08:24:33   =========================== short test summary info ============================
08:24:33   ERROR test/Services/test_fileoperations.py - sqlalchemy.exc.OperationalError: (pyodbc.OperationalError) ('HYT00', '[HYT00] [Microsoft][ODBC Driver 17 for SQL Server]Login timeout expired (0) (SQLDriverConnect)')
08:24:33   (Background on this error at: http://sqlalche.me/e/13/e3q8)
08:24:33   !!!!!!!!!!!!!!!!!!!! Interrupted: 1 error during collection !!!!!!!!!!!!!!!!!!!!
08:24:33   ======================== 10 warnings, 1 error in 33.97s ========================
08:24:34   Process exited with code 2
08:24:34   Process exited with code 2 (Step: PYTHON PYTEST : LINUX (Command Line))
08:24:34   Step PYTHON PYTEST : LINUX (Command Line) failed
08:24:34 Step 8/16: PYTHON PYTEST : WINDOWS (PowerShell)
08:24:34 Step 9/16: SONARQUBE (SonarQube Runner)
08:24:34 Step 10/16: SONAR QUALITY GATE CHECK (PowerShell)
08:24:34 Step 11/16: SONAR QUALITY GATE CHECK : LINUX (PowerShell)
08:24:34 Step 12/16: PYTHON AND PIP VERSION (Command Line)
08:24:34 Step 13/16: PIP INSTALL (Command Line)
08:24:34 Step 14/16: CREATE LOGS FOLDER (PowerShell)
08:24:34 Step 15/16: PYTHON PYTEST (Command Line)
08:24:34 Step 16/16: SONAR QUALITY GATE CHECK (PowerShell)
08:24:34 Publishing internal artifacts
08:24:35 Build is failed. Artifacts will not be published for this build
08:24:35 Build finished
