# test/Services/test_fileoperations.py
import unittest
from unittest.mock import MagicMock, patch, call
import os
import datetime
import shutil
import sys

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
    spec=True # Good practice: ensures the mock instance has the right methods
)

# 3. START the patch *before* the problematic import happens.
MockDbClass = patcher_db_class.start()

# 4. NOW import the module under test.
#    When Python executes 'dbops_obj = dbops.dboperations()' inside fileoperations.py,
#    'dbops.dboperations' is actually our MockDbClass.
#    Calling MockDbClass() returns 'mock_db_instance_for_import'.
#    Therefore, fo.dbops_obj will be assigned our 'mock_db_instance_for_import'.
#    Crucially, the real __init__ and the engine.connect() call are bypassed.
import Services.fileoperations as fo
import globalvars as gvar
from Services.CustomException import FileValidationException # Import exceptions if needed

# --- Test Class Definition ---

class Test_FileOperations(unittest.TestCase):

    @classmethod
    def tearDownClass(cls):
        # Stop the patcher we started globally for the module import
        # Ensures the patch doesn't leak outside this test module if run in sequence
        patcher_db_class.stop()

    def setUp(self):
        # The database object (fo.dbops_obj) is ALREADY mocked by the pre-import patch.
        # We just need to get a reference to it and configure it for each test.
        self.mock_dbops_obj = fo.dbops_obj

        # Reset the mock's state and configure return values for this specific test
        self.mock_dbops_obj.reset_mock() # Good practice to avoid state leaking between tests
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
        self.mock_dbops_obj.insert_actionLog.return_value = None

        # Configure global variables (using the mocked methods)
        gvar.sadrd_settings = self.mock_dbops_obj.SadrdSysSettings()
        gvar.sadrd_ErrMessages = self.mock_dbops_obj.SADRD_Sys_Message()
        gvar.user_id = 'test_user_123'

        # Patch OS/shutil functions - these are fine to patch here per-test
        self.patcher_exists = patch('os.path.exists', return_value=True)
        self.patcher_makedirs = patch('os.makedirs')
        self.patcher_listdir = patch('os.listdir')
        self.patcher_rename = patch('os.rename', return_value=None)
        self.patcher_copyfile = patch('shutil.copyfile')
        self.patcher_os_path_join = patch('os.path.join', side_effect=os.path.join)
        # Patch is_open *within* fileoperations, as it's likely defined/used there
        self.patcher_is_open = patch('Services.fileoperations.is_open', return_value=False)

        self.mock_exists = self.patcher_exists.start()
        self.mock_makedirs = self.patcher_makedirs.start()
        self.mock_listdir = self.patcher_listdir.start()
        self.mock_rename = self.patcher_rename.start()
        self.mock_copyfile = self.patcher_copyfile.start()
        self.mock_os_path_join = self.patcher_os_path_join.start()
        self.mock_is_open = self.patcher_is_open.start()

        # Add self instances to manage these patches started in setUp
        self.addCleanup(self.patcher_exists.stop)
        self.addCleanup(self.patcher_makedirs.stop)
        self.addCleanup(self.patcher_listdir.stop)
        self.addCleanup(self.patcher_rename.stop)
        self.addCleanup(self.patcher_copyfile.stop)
        self.addCleanup(self.patcher_os_path_join.stop)
        self.addCleanup(self.patcher_is_open.stop)


    # tearDown is not strictly needed if using addCleanup in setUp for os/shutil patches
    # def tearDown(self):
    #     patch.stopall() # Be cautious if using stopall with class-level patches

    def _build_path(self, *args):
        # Use the actual os.path.join unless specifically mocking its side effect
        return os.path.join(*args) # Or self.mock_os_path_join if you need to track calls

    # --- Your test methods remain largely the same ---
    # They will now interact with the pre-configured self.mock_dbops_obj (fo.dbops_obj)

    def test_getinpfilenames_toprocess(self):
        folderpath = 'test_folder'; inpLoadFolder = 'input_files'
        full_dir_path = self._build_path(folderpath, inpLoadFolder)
        self.mock_listdir.return_value = ['file1.xlsx', 'file2.xlsm', 'file3.csv', 'file4.txt']
        expected_files = [ self._build_path(full_dir_path, f) for f in ['file1.xlsx', 'file2.xlsm', 'file3.csv']]

        # **** Critical check: Ensure fo.dbops_obj is the mock ****
        self.assertIs(fo.dbops_obj, self.mock_dbops_obj)

        result = fo.getinpfilenames_toprocess(folderpath, inpLoadFolder)
        # os.path.join might be called multiple times, use assert_any_call or check call list
        # self.mock_os_path_join.assert_any_call(folderpath, inpLoadFolder)
        # self.mock_os_path_join.assert_any_call(full_dir_path, 'file1.xlsx')
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
        # Be careful with string concatenation vs os.path.join for paths
        expected_dest_copyfile = os.path.join(local_path, filename) # More robust
        expected_src_copyfile = os.path.join(server_path, filename) # More robust
        self.assertEqual(result, [expected_dest_os_join])
        self.mock_is_open.assert_called_once_with(expected_src_os_join)
        self.mock_copyfile.assert_called_once_with(expected_src_copyfile, expected_dest_copyfile)
        # *** Assert against the mock instance ***
        self.mock_dbops_obj.insert_actionLog.assert_called_once()

    # ... (adapt other tests similarly, ensuring assertions use self.mock_dbops_obj) ...

    def test_Downloadfilenames_toprocess_wrong_file_type_schd(self):
        server_path, local_path, action, year = 's', 'l', 'Annual Stmt - Sch D', '2023'
        filename = '2023JHUSASchD.xlsx'
        self.mock_listdir.return_value = [filename]
        with self.assertRaises(FileValidationException):
            fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
        expected_src_os_join = self._build_path(server_path, filename)
        self.mock_is_open.assert_called_once_with(expected_src_os_join)
        # *** Assert against the mock instance ***
        self.mock_dbops_obj.BuildErrorMessage.assert_called_once_with(f'E003,{filename},None; E002,,')
        self.mock_copyfile.assert_not_called()
        self.mock_dbops_obj.insert_actionLog.assert_called_once() # Check if actionLog is called even on failure

    # ... etc ...

if __name__ == '__main__':
    unittest.main()

