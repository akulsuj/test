# test/Services/test_fileoperations.py
import unittest
from unittest.mock import MagicMock, patch, call
import os
import datetime
import shutil
import sys

# Assume the following files exist:
# Services/fileoperations.py (cannot be modified, imports dboperations and instantiates at module level)
# Services/dboperations.py (contains the dboperations class with __init__ that connects)
# Services/CustomException.py (defines FileValidationException)
# globalvars.py (contains global variables like user_id, sadrd_settings, etc.)

# --- IMPORTANT: Patching Setup BEFORE importing the module under test ---

# 1. Define the mock instance that will be returned *instead* of a real dbops instance.
#    We need to configure this instance later in setUp.
mock_db_instance_for_import = MagicMock()

# 2. Create a patcher for the dboperations class in its original location.
#    This patch will replace the actual class with a mock object/factory.
#    When the patched class is called (like a function), it will return
#    our predefined mock_db_instance_for_import.
patcher_db_class = patch(
    'Services.dboperations.dboperations',  # Target the class where it's defined
    return_value=mock_db_instance_for_import,
    spec=True  # Good practice: ensures the mock instance has the right methods/attributes
)

# 3. START the patch *before* the problematic import happens.
MockDbClass = patcher_db_class.start()

# 4. NOW import the module under test and related modules.
#    When Python executes 'dbops_obj = dbops.dboperations()' inside fileoperations.py,
#    'dbops.dboperations' is actually our MockDbClass.
#    Calling MockDbClass() returns 'mock_db_instance_for_import'.
#    Therefore, fo.dbops_obj will be assigned our 'mock_db_instance_for_import'.
#    Crucially, the real __init__ and the engine.connect() call are bypassed.
import Services.fileoperations as fo
import globalvars as gvar
from Services.CustomException import FileValidationException

# --- Test Class Definition ---

class Test_FileOperations(unittest.TestCase):

    @classmethod
    def tearDownClass(cls):
        """
        Stop the patcher we started globally for the module import.
        Ensures the patch doesn't leak outside this test module.
        """
        patcher_db_class.stop()

    def setUp(self):
        """
        Set up test fixtures for each test method.
        """
        # The database object (fo.dbops_obj) is ALREADY mocked by the pre-import patch.
        # We just need to get a reference to it and configure it for each test.
        self.mock_dbops_obj = fo.dbops_obj

        # Reset the mock's state and configure return values for this specific test
        # Prevents state (like call counts) leaking between tests.
        self.mock_dbops_obj.reset_mock()

        # --- Configure mock database object behavior ---
        self.mock_dbops_obj.BuildErrorMessage.return_value = "Mocked Error Message"
        # Simulate settings fetched from the database
        mock_settings = [
            MagicMock(settingName="Valid_Company", settingValue="JHUSA"),
            MagicMock(settingName="Valid_Company", settingValue="JHIL"),
            MagicMock(settingName="Valid_Quarter", settingValue="Q1"),
            MagicMock(settingName="Valid_Quarter", settingValue="Q2"),
            MagicMock(settingName="Valid_Quarter", settingValue="Q3"),
            MagicMock(settingName="Valid_Quarter", settingValue="Q4"),
            MagicMock(settingName="Filename_AnnStmtSchD", settingValue="SchD"),
            MagicMock(settingName="Filename_QualFTC", settingValue="QualFTC"),
            MagicMock(settingName="Filename_FTCGrossup", settingValue="FTCGrossup"),
        ]
        self.mock_dbops_obj.SadrdSysSettings.return_value = mock_settings
        self.mock_dbops_obj.SADRD_Sys_Message.return_value = MagicMock() # Mock message object if needed
        self.mock_dbops_obj.insert_actionLog.return_value = None # Simulate successful log insertion

        # Configure global variables (as if populated via mocked db calls)
        # Ensure the code under test uses these mocked globals
        gvar.sadrd_settings = self.mock_dbops_obj.SadrdSysSettings()
        gvar.sadrd_ErrMessages = self.mock_dbops_obj.SADRD_Sys_Message()
        gvar.user_id = 'test_user_123' # Set a dummy user ID for tests

        # --- Patch OS and shutil functions ---
        # These are typically patched per-test using setUp/tearDown or addCleanup
        self.patcher_exists = patch('os.path.exists', return_value=True)
        self.patcher_makedirs = patch('os.makedirs')
        self.patcher_listdir = patch('os.listdir')
        self.patcher_rename = patch('os.rename', return_value=None)
        self.patcher_copyfile = patch('shutil.copyfile')
        # Use side_effect for os.path.join to keep its real functionality but allow tracking if needed
        self.patcher_os_path_join = patch('os.path.join', side_effect=os.path.join)
        # Patch is_open function presumably located within fileoperations module
        # Adjust the target string if 'is_open' is imported differently in fileoperations.py
        self.patcher_is_open = patch('Services.fileoperations.is_open', return_value=False)

        # Start the os/shutil patches and get mock references
        self.mock_exists = self.patcher_exists.start()
        self.mock_makedirs = self.patcher_makedirs.start()
        self.mock_listdir = self.patcher_listdir.start()
        self.mock_rename = self.patcher_rename.start()
        self.mock_copyfile = self.patcher_copyfile.start()
        self.mock_os_path_join = self.patcher_os_path_join.start()
        self.mock_is_open = self.patcher_is_open.start()

        # Use addCleanup to automatically stop these patches after each test
        self.addCleanup(self.patcher_exists.stop)
        self.addCleanup(self.patcher_makedirs.stop)
        self.addCleanup(self.patcher_listdir.stop)
        self.addCleanup(self.patcher_rename.stop)
        self.addCleanup(self.patcher_copyfile.stop)
        self.addCleanup(self.patcher_os_path_join.stop)
        self.addCleanup(self.patcher_is_open.stop)

    def _build_path(self, *args):
        """Helper to construct paths consistently using os.path.join."""
        # This uses the real os.path.join because of the side_effect in the patch
        return os.path.join(*args)

    # --- Test Methods ---
    # All assertions involving the database object should use self.mock_dbops_obj

    def test_getinpfilenames_toprocess(self):
        folderpath = 'test_folder'
        inpLoadFolder = 'input_files'
        full_dir_path = self._build_path(folderpath, inpLoadFolder)
        self.mock_listdir.return_value = ['file1.xlsx', 'file2.xlsm', 'file3.csv', 'file4.txt']
        expected_files = [ self._build_path(full_dir_path, f) for f in ['file1.xlsx', 'file2.xlsm', 'file3.csv']]

        # Ensure the db object in the module under test IS our mock
        self.assertIs(fo.dbops_obj, self.mock_dbops_obj)

        result = fo.getinpfilenames_toprocess(folderpath, inpLoadFolder)

        # Check that os.path.join was called (implicitly via _build_path or directly in fo)
        # Check that listdir was called correctly
        self.mock_listdir.assert_called_once_with(full_dir_path)

        # Check the results
        self.assertEqual(sorted(result), sorted(expected_files))
        self.assertNotIn(self._build_path(full_dir_path, 'file4.txt'), result)


    def test_Downloadfilenames_toprocess_annual_schd_success(self):
        server_path, local_path, action, year = 's_path', 'l_path', 'Annual Stmt - Sch D', '2023'
        filename = '2023JHUSASchD.csv'
        self.mock_listdir.return_value = [filename] # Simulate os.listdir finding this file

        result = fo.Downloadfilenames_toprocess(server_path, local_path, action, year)

        expected_dest_path = self._build_path(local_path, filename)
        expected_src_path = self._build_path(server_path, filename)

        self.assertEqual(result, [expected_dest_path]) # Check returned list
        self.mock_is_open.assert_called_once_with(expected_src_path) # Check if file lock check was done
        self.mock_copyfile.assert_called_once_with(expected_src_path, expected_dest_path) # Check copy call
        self.mock_dbops_obj.insert_actionLog.assert_called_once() # Check DB log call


    def test_Downloadfilenames_toprocess_qualftc_success(self):
        server_path, local_path, action, year = 's_path', 'l_path', 'QualPctFTC', '2023'
        filename = '2023QualFTC.xlsx'
        self.mock_listdir.return_value = [filename]

        result = fo.Downloadfilenames_toprocess(server_path, local_path, action, year)

        expected_dest_path = self._build_path(local_path, filename)
        expected_src_path = self._build_path(server_path, filename)

        self.assertEqual(result, [expected_dest_path])
        self.mock_is_open.assert_called_once_with(expected_src_path)
        self.mock_copyfile.assert_called_once_with(expected_src_path, expected_dest_path)
        self.mock_dbops_obj.insert_actionLog.assert_called_once()


    def test_Downloadfilenames_toprocess_ftcgrossup_success(self):
        server_path, local_path, action, year = 's_path', 'l_path', 'FTCGrossup', '2023'
        files = [f'2023_Q{i}_FTCGrossup.xlsx' for i in range(1, 5)]
        self.mock_listdir.return_value = files

        result = fo.Downloadfilenames_toprocess(server_path, local_path, action, year)

        expected_dest_files = [self._build_path(local_path, f) for f in files]
        expected_src_files = [self._build_path(server_path, f) for f in files]
        expected_is_open_calls = [call(src) for src in expected_src_files]
        expected_copy_calls = [call(src, dest) for src, dest in zip(expected_src_files, expected_dest_files)]

        self.assertEqual(sorted(result), sorted(expected_dest_files))
        # Use assert_has_calls for checking multiple calls, any_order=True if order doesn't matter
        self.mock_is_open.assert_has_calls(expected_is_open_calls, any_order=True)
        self.assertEqual(self.mock_is_open.call_count, 4)
        self.mock_copyfile.assert_has_calls(expected_copy_calls, any_order=True)
        self.assertEqual(self.mock_copyfile.call_count, 4)
        self.mock_dbops_obj.insert_actionLog.assert_called_once()


    def test_Downloadfilenames_toprocess_wrong_file_type_schd(self):
        server_path, local_path, action, year = 's_path', 'l_path', 'Annual Stmt - Sch D', '2023'
        filename = '2023JHUSASchD.xlsx' # Wrong extension for SchD
        self.mock_listdir.return_value = [filename]

        with self.assertRaises(FileValidationException):
            fo.Downloadfilenames_toprocess(server_path, local_path, action, year)

        expected_src_path = self._build_path(server_path, filename)
        self.mock_is_open.assert_called_once_with(expected_src_path) # Check if it still checked lock
        # Check that the correct error message was built
        self.mock_dbops_obj.BuildErrorMessage.assert_called_once_with(f'E003,{filename},None; E002,,')
        self.mock_copyfile.assert_not_called() # Ensure file was not copied
        self.mock_dbops_obj.insert_actionLog.assert_called_once() # Check log was called (even for failure)


    def test_Downloadfilenames_toprocess_file_open(self):
        server_path, local_path, action, year = 's_path', 'l_path', 'Annual Stmt - Sch D', '2023'
        filename = '2023JHUSASchD.csv'
        self.mock_listdir.return_value = [filename]
        self.mock_is_open.return_value = True # Simulate file being open/locked

        with self.assertRaises(FileValidationException):
            fo.Downloadfilenames_toprocess(server_path, local_path, action, year)

        expected_src_path = self._build_path(server_path, filename)
        self.mock_is_open.assert_called_once_with(expected_src_path) # Verify lock check happened
        self.mock_dbops_obj.BuildErrorMessage.assert_called_once_with(f'E006,{filename},None; E002,,')
        self.mock_copyfile.assert_not_called() # Ensure file was not copied
        self.mock_dbops_obj.insert_actionLog.assert_called_once() # Check log


    @patch('Services.fileoperations.getinpfilenames_toprocess') # Patch function within fileoperations
    def test_ExtractFilesToLoad(self, mock_getinp):
        folderpath, action = 'local_folder', 'SomeAction'
        # Use os.path.join for realistic paths
        expected_list = [os.path.join('local_folder', 'f1.csv'), os.path.join('local_folder', 'f2.xlsx')]
        mock_getinp.return_value = expected_list # Configure the mock for this specific function

        result = fo.ExtractFilesToLoad(folderpath, action)

        mock_getinp.assert_called_once_with(folderpath, "") # Check how the patched function was called
        self.assertEqual(result, expected_list)


    @patch('Services.fileoperations.Downloadfilenames_toprocess') # Patch function within fileoperations
    def test_DownloadServerFilesToLoad(self, mock_download):
        server_path, local_path, action, year = 's_path', 'l_path', 'ActionX', '2024'
        expected_list = [os.path.join(local_path, 'file1.csv')]
        mock_download.return_value = expected_list # Configure the mock for this specific function

        result = fo.DownloadServerFilesToLoad(server_path, local_path, action, year)

        mock_download.assert_called_once_with(server_path, local_path, action, year)
        self.assertEqual(result, expected_list)


    def test_Downloadfilenames_toprocess_dest_dir_does_not_exist(self):
        server_path, local_path, action, year = 's_path', 'l_path_new', 'Annual Stmt - Sch D', '2023'
        filename = '2023JHUSASchD.csv'
        # Simulate destination not existing initially, then existing after makedirs
        self.mock_exists.side_effect = [False, True]
        self.mock_listdir.return_value = [filename]

        fo.Downloadfilenames_toprocess(server_path, local_path, action, year)

        # Check that existence was checked and directory was created
        self.mock_exists.assert_called_once_with(local_path)
        self.mock_makedirs.assert_called_once_with(local_path)
        # Check other calls happened as expected
        expected_src_path = self._build_path(server_path, filename)
        expected_dest_path = self._build_path(local_path, filename)
        self.mock_is_open.assert_called_once_with(expected_src_path)
        self.mock_copyfile.assert_called_once_with(expected_src_path, expected_dest_path)
        self.mock_dbops_obj.insert_actionLog.assert_called_once()


    def test_Downloadfilenames_toprocess_ignores_temp_files(self):
        server_path, local_path, action, year = 's_path', 'l_path', 'Annual Stmt - Sch D', '2023'
        good_file, temp_file = '2023JHUSASchD.csv', '~$2023JHUSASchD.csv'
        self.mock_listdir.return_value = [temp_file, good_file] # List both files

        result = fo.Downloadfilenames_toprocess(server_path, local_path, action, year)

        expected_dest_path = self._build_path(local_path, good_file)
        expected_src_path = self._build_path(server_path, good_file)

        self.assertEqual(result, [expected_dest_path]) # Only the good file should be returned
        # is_open and copyfile should only be called for the non-temp file
        self.mock_is_open.assert_called_once_with(expected_src_path)
        self.mock_copyfile.assert_called_once_with(expected_src_path, expected_dest_path)
        self.mock_dbops_obj.insert_actionLog.assert_called_once()


    def test_Downloadfilenames_toprocess_ignores_wrong_type_or_extension_filter(self):
        server_path, local_path, action, year = 's_path', 'l_path', 'Annual Stmt - Sch D', '2023'
        good_file='2023JHUSASchD.csv'; wrong_type='2023WrongCompanySchD.csv'; wrong_ext='2023JHUSASchD.txt'
        self.mock_listdir.return_value = [wrong_type, wrong_ext, good_file]

        result = fo.Downloadfilenames_toprocess(server_path, local_path, action, year)

        expected_dest_path = self._build_path(local_path, good_file)
        expected_src_path = self._build_path(server_path, good_file)

        self.assertEqual(result, [expected_dest_path]) # Only the correctly named file processed
        self.mock_is_open.assert_called_once_with(expected_src_path)
        self.mock_copyfile.assert_called_once_with(expected_src_path, expected_dest_path)
        self.mock_dbops_obj.insert_actionLog.assert_called_once()


    def test_Downloadfilenames_toprocess_empty_after_initial_filter(self):
        server_path, local_path, action, year = 's_path', 'l_path', 'Annual Stmt - Sch D', '2023'
        # Files that should be ignored by initial filters (temp, wrong ext, wrong name pattern)
        self.mock_listdir.return_value = ['~$f.csv', 'f.txt', 'WrongFilename.xlsx', '2023WrongCompanySchD.csv']

        with self.assertRaises(FileValidationException):
            fo.Downloadfilenames_toprocess(server_path, local_path, action, year)

        # Check that the "No files found" error message was built
        self.mock_dbops_obj.BuildErrorMessage.assert_called_once_with('E002,,')
        self.mock_copyfile.assert_not_called()
        self.mock_is_open.assert_not_called()
        self.mock_dbops_obj.insert_actionLog.assert_called_once() # Log should still be called


    def test_Downloadfilenames_toprocess_wrong_extension_qualftc(self):
        server_path, local_path, action, year = 's_path', 'l_path', 'QualPctFTC', '2023'
        filename = '2023QualFTC.csv' # Wrong extension for QualFTC
        self.mock_listdir.return_value = [filename]

        with self.assertRaises(FileValidationException):
            fo.Downloadfilenames_toprocess(server_path, local_path, action, year)

        expected_src_path = self._build_path(server_path, filename)
        self.mock_is_open.assert_called_once_with(expected_src_path)
        self.mock_dbops_obj.BuildErrorMessage.assert_called_once_with(f'E003,{filename},None; E002,,')
        self.mock_copyfile.assert_not_called()
        self.mock_dbops_obj.insert_actionLog.assert_called_once()


    def test_Downloadfilenames_toprocess_wrong_extension_ftcgrossup(self):
        server_path, local_path, action, year = 's_path', 'l_path', 'FTCGrossup', '2023'
        filename = '2023_Q1_FTCGrossup.csv' # Wrong extension for FTCGrossup
        self.mock_listdir.return_value = [filename]

        with self.assertRaises(FileValidationException):
            fo.Downloadfilenames_toprocess(server_path, local_path, action, year)

        expected_src_path = self._build_path(server_path, filename)
        # Depending on implementation, is_open might be called before extension check
        # self.mock_is_open.assert_called_once_with(expected_src_path) # Uncomment if applicable
        # Or maybe the extension check happens earlier
        self.mock_is_open.assert_not_called() # More likely if extension checked first
        self.mock_dbops_obj.BuildErrorMessage.assert_called_once_with(f'E003,{filename},None; E002,,')
        self.mock_copyfile.assert_not_called()
        self.mock_dbops_obj.insert_actionLog.assert_called_once()

# --- Allow running tests directly ---
if __name__ == '__main__':
    unittest.main()
