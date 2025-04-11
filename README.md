

import unittest
from unittest.mock import MagicMock, patch, call
import os

# Assume Services.dboperations, Services.CustomException, globalvars exist

mock_db_instance_for_import = MagicMock()
patcher_db_class = patch(
    'Services.dboperations.dboperations',
    return_value=mock_db_instance_for_import,
    spec=True
)
MockDbClass = patcher_db_class.start()

import Services.fileoperations as fo
import globalvars as gvar
from Services.CustomException import FileValidationException

# --- Helper Function to Force Forward Slashes (used only in comparison) ---
def force_forward_slashes(path):
    """Converts all backslashes to forward slashes in a path string."""
    if isinstance(path, str):
        return path.replace('\\', '/')
    return path

class Test_FileOperations(unittest.TestCase):

    @classmethod
    def tearDownClass(cls):
        patcher_db_class.stop()

    def setUp(self):
        self.mock_dbops_obj = fo.dbops_obj
        self.mock_dbops_obj.reset_mock()

        self.mock_dbops_obj.BuildErrorMessage.return_value = "Mocked Error Message"
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
        self.mock_dbops_obj.SADRD_Sys_Message.return_value = MagicMock()
        self.mock_dbops_obj.insert_actionLog.return_value = None

        gvar.sadrd_settings = self.mock_dbops_obj.SadrdSysSettings()
        gvar.sadrd_ErrMessages = self.mock_dbops_obj.SADRD_Sys_Message()
        gvar.user_id = 'test_user_123'

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

        self.addCleanup(self.patcher_exists.stop)
        self.addCleanup(self.patcher_makedirs.stop)
        self.addCleanup(self.patcher_listdir.stop)
        self.addCleanup(self.patcher_rename.stop)
        self.addCleanup(self.patcher_copyfile.stop)
        self.addCleanup(self.patcher_os_path_join.stop)
        self.addCleanup(self.patcher_is_open.stop)

    # MODIFIED _build_path - Reverted to only use os.path.join
    def _build_path(self, *args):
        """Generate expected paths using os.path.join for consistency."""
        return os.path.join(*args) # Use native path joining

    # assertPathCalledWith still normalizes AND forces forward slashes for comparison consistency
    def assertPathCalledWith(self, mock_func, expected_args_tuple):
        """Asserts mock_func was called once with normalized path args (forced '/')."""
        try:
            mock_func.assert_called_once()
        except AssertionError as e:
            raise AssertionError(
                f"Expected '{mock_func._extract_mock_name()}' to be called once. "
                f"Actual calls: {mock_func.call_count}\n"
                f"Call args list: {mock_func.call_args_list}\nOriginal Error: {e}"
            )

        actual_args, actual_kwargs = mock_func.call_args
        if len(actual_args) != len(expected_args_tuple):
             raise AssertionError(
                 f"Expected {len(expected_args_tuple)} positional arguments for '{mock_func._extract_mock_name()}', "
                 f"but got {len(actual_args)}. Actual args: {actual_args}"
            )

        # Normalize with normpath AND force forward slashes for comparison
        normalized_actual = tuple(force_forward_slashes(os.path.normpath(arg)) if isinstance(arg, str) else arg for arg in actual_args)
        # Ensure expected args are also fully normalized for comparison
        normalized_expected = tuple(force_forward_slashes(os.path.normpath(arg)) if isinstance(arg, str) else arg for arg in expected_args_tuple)

        self.assertEqual(normalized_actual, normalized_expected,
                         f"Normalized call args differ (forced '/' separators) for '{mock_func._extract_mock_name()}':\n"
                         f"  Actual: {normalized_actual}\n"
                         f"Expected: {normalized_expected}")

    # assertPathHasCalls still normalizes AND forces forward slashes for comparison consistency
    def assertPathHasCalls(self, mock_func, expected_calls_list, any_order=False):
        """Asserts mock_func has calls matching expected_calls_list (normalized, forced '/')."""
        actual_calls = mock_func.call_args_list
        mock_name = mock_func._extract_mock_name()

        def normalize_call_args_forced_slash(call_obj):
            try:
                args, kwargs = call_obj
                if not hasattr(args, '__iter__'):
                     raise TypeError(f"Args part of call object is not iterable: {args!r}")
                # Normalize with normpath AND force forward slashes
                normalized_args = tuple(force_forward_slashes(os.path.normpath(arg)) if isinstance(arg, str) else arg for arg in args)
                return call(*normalized_args, **kwargs)
            except (ValueError, TypeError) as e:
                raise TypeError(f"Failed to unpack/normalize call object: {call_obj!r}. Original error: {e}")

        try:
            normalized_expected_calls = [normalize_call_args_forced_slash(c) for c in expected_calls_list]
            normalized_actual_calls = [normalize_call_args_forced_slash(c) for c in actual_calls]
        except TypeError as e:
             raise AssertionError(f"Error normalizing call lists for '{mock_name}': {e}")

        if any_order:
            try:
                actual_set = set(map(str, normalized_actual_calls))
                expected_set = set(map(str, normalized_expected_calls))
            except Exception as e:
                 raise AssertionError(f"Failed converting normalized calls to strings for set comparison for '{mock_name}': {e}")

            if actual_set != expected_set:
                 missing_calls = expected_set - actual_set
                 unexpected_calls = actual_set - expected_set
                 raise AssertionError(
                     f"Normalized calls (forced '/', any order) differ for '{mock_name}':\n"
                     f"--- Missing ---\n{missing_calls}\n"
                     f"--- Unexpected ---\n{unexpected_calls}\n"
                     f"--- Actual (Normalized) ---\n{normalized_actual_calls}\n"
                     f"--- Expected (Normalized) ---\n{normalized_expected_calls}"
                 )
        else:
             self.assertEqual(normalized_actual_calls, normalized_expected_calls,
                              f"Normalized calls (forced '/', order matters) differ for '{mock_name}':\n"
                              f"  Actual: {normalized_actual_calls}\n"
                              f"Expected: {normalized_expected_calls}")


    # --- Test Methods ---

    def test_getinpfilenames_toprocess(self):
        folderpath = 'test_folder'; inpLoadFolder = 'input_files'
        # Use reverted _build_path for native path generation
        full_dir_path = self._build_path(folderpath, inpLoadFolder)
        self.mock_listdir.return_value = ['file1.xlsx', 'file2.xlsm', 'file3.csv', 'file4.txt']
        # Expected files also use native path generation
        expected_files = [ self._build_path(full_dir_path, f) for f in ['file1.xlsx', 'file2.xlsm', 'file3.csv']]
        self.assertIs(fo.dbops_obj, self.mock_dbops_obj)
        result = fo.getinpfilenames_toprocess(folderpath, inpLoadFolder)
        # Check listdir was called with the natively generated path
        self.mock_listdir.assert_called_once_with(full_dir_path)
        # Compare results - apply normalization+slash forcing only for comparison consistency
        self.assertEqual(sorted([force_forward_slashes(os.path.normpath(p)) for p in result]),
                         sorted([force_forward_slashes(os.path.normpath(p)) for p in expected_files]))
        # Check exclusion using natively generated path
        self.assertNotIn(self._build_path(full_dir_path, 'file4.txt'), result)


    # Other tests remain unchanged from Plan 6.0 / Final version of previous response
    # They use the assertPath... helpers which handle the normalization + slash forcing
    # internally, or use manual checks which also force slashes for comparison.

    def test_Downloadfilenames_toprocess_annual_schd_success(self):
        server_path, local_path, action, year = 's_path', 'l_path', 'Annual Stmt - Sch D', '2023'
        filename = '2023JHUSASchD.csv'
        self.mock_listdir.return_value = [filename]
        result = fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
        expected_dest_path = self._build_path(local_path, filename)
        expected_src_path = self._build_path(server_path, filename)
        self.assertEqual([force_forward_slashes(p) for p in result], [force_forward_slashes(expected_dest_path)])
        self.assertPathCalledWith(self.mock_is_open, (expected_src_path,))
        self.assertPathCalledWith(self.mock_copyfile, (expected_src_path, expected_dest_path))
        self.mock_dbops_obj.insert_actionLog.assert_called_once()

    def test_Downloadfilenames_toprocess_qualftc_success(self):
        server_path, local_path, action, year = 's_path', 'l_path', 'QualPctFTC', '2023'
        filename = '2023QualFTC.xlsx'
        self.mock_listdir.return_value = [filename]
        result = fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
        expected_dest_path = self._build_path(local_path, filename)
        expected_src_path = self._build_path(server_path, filename)
        self.assertEqual([force_forward_slashes(p) for p in result], [force_forward_slashes(expected_dest_path)])
        self.assertPathCalledWith(self.mock_is_open, (expected_src_path,))
        self.assertPathCalledWith(self.mock_copyfile, (expected_src_path, expected_dest_path))
        self.mock_dbops_obj.insert_actionLog.assert_called_once()

    def test_Downloadfilenames_toprocess_ftcgrossup_success(self):
        server_path, local_path, action, year = 's_path', 'l_path', 'FTCGrossup', '2023'
        files = [f'2023_Q{i}_FTCGrossup.xlsx' for i in range(1, 5)]
        self.mock_listdir.return_value = files
        result = fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
        expected_dest_files = [self._build_path(local_path, f) for f in files]
        expected_src_files = [self._build_path(server_path, f) for f in files]
        self.assertEqual(sorted([force_forward_slashes(p) for p in result]),
                         sorted([force_forward_slashes(p) for p in expected_dest_files]))

        self.assertEqual(self.mock_is_open.call_count, 4, "is_open call count mismatch")
        try:
            actual_is_open_args = {force_forward_slashes(os.path.normpath(c.args[0])) for c in self.mock_is_open.call_args_list}
        except (AttributeError, IndexError, TypeError) as e:
             raise AssertionError(f"Could not extract args from mock_is_open.call_args_list: {e}\nList: {self.mock_is_open.call_args_list}")
        expected_is_open_args = {force_forward_slashes(os.path.normpath(p)) for p in expected_src_files}
        self.assertSetEqual(actual_is_open_args, expected_is_open_args, "is_open call arguments mismatch")

        self.assertEqual(self.mock_copyfile.call_count, 4, "copyfile call count mismatch")
        try:
             actual_copyfile_args = {
                 (force_forward_slashes(os.path.normpath(c.args[0])), force_forward_slashes(os.path.normpath(c.args[1])))
                 for c in self.mock_copyfile.call_args_list
             }
        except (AttributeError, IndexError, TypeError) as e:
              raise AssertionError(f"Could not extract args from mock_copyfile.call_args_list: {e}\nList: {self.mock_copyfile.call_args_list}")
        expected_copyfile_args = {
            (force_forward_slashes(os.path.normpath(src)), force_forward_slashes(os.path.normpath(dest)))
            for src, dest in zip(expected_src_files, expected_dest_files)
        }
        self.assertSetEqual(actual_copyfile_args, expected_copyfile_args, "copyfile call arguments mismatch")
        self.mock_dbops_obj.insert_actionLog.assert_called_once()

    def test_Downloadfilenames_toprocess_wrong_file_type_schd(self):
        server_path, local_path, action, year = 's_path', 'l_path', 'Annual Stmt - Sch D', '2023'
        filename = '2023JHUSASchD.xlsx'
        self.mock_listdir.return_value = [filename]
        with self.assertRaises(FileValidationException):
            fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
        expected_src_path = self._build_path(server_path, filename)
        self.assertPathCalledWith(self.mock_is_open, (expected_src_path,))
        self.mock_dbops_obj.BuildErrorMessage.assert_called_once_with(f'E003,{filename},None; E002,,')
        self.mock_copyfile.assert_not_called()
        self.mock_dbops_obj.insert_actionLog.assert_called_once()

    def test_Downloadfilenames_toprocess_file_open(self):
        server_path, local_path, action, year = 's_path', 'l_path', 'Annual Stmt - Sch D', '2023'
        filename = '2023JHUSASchD.csv'
        self.mock_listdir.return_value = [filename]
        self.mock_is_open.return_value = True
        with self.assertRaises(FileValidationException):
            fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
        expected_src_path = self._build_path(server_path, filename)
        self.assertPathCalledWith(self.mock_is_open, (expected_src_path,))
        self.mock_dbops_obj.BuildErrorMessage.assert_called_once_with(f'E006,{filename},None; E002,,')
        self.mock_copyfile.assert_not_called()
        self.mock_dbops_obj.insert_actionLog.assert_called_once()

    @patch('Services.fileoperations.getinpfilenames_toprocess')
    def test_ExtractFilesToLoad(self, mock_getinp):
        folderpath, action = 'local_folder', 'SomeAction'
        expected_list = [os.path.join('local_folder', 'f1.csv'), os.path.join('local_folder', 'f2.xlsx')]
        mock_getinp.return_value = expected_list
        result = fo.ExtractFilesToLoad(folderpath, action)
        mock_getinp.assert_called_once_with(folderpath, "")
        self.assertEqual(result, expected_list)

    @patch('Services.fileoperations.Downloadfilenames_toprocess')
    def test_DownloadServerFilesToLoad(self, mock_download):
        server_path, local_path, action, year = 's_path', 'l_path', 'ActionX', '2024'
        expected_list = [os.path.join(local_path, 'file1.csv')]
        mock_download.return_value = expected_list
        result = fo.DownloadServerFilesToLoad(server_path, local_path, action, year)
        mock_download.assert_called_once_with(server_path, local_path, action, year)
        self.assertEqual(result, expected_list)

    def test_Downloadfilenames_toprocess_dest_dir_does_not_exist(self):
        server_path, local_path, action, year = 's_path', 'l_path_new', 'Annual Stmt - Sch D', '2023'
        filename = '2023JHUSASchD.csv'
        self.mock_exists.side_effect = lambda p: False if p == local_path else True
        self.mock_listdir.return_value = [filename]
        fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
        self.mock_exists.assert_any_call(local_path)
        self.mock_makedirs.assert_called_once_with(local_path)
        expected_src_path = self._build_path(server_path, filename)
        expected_dest_path = self._build_path(local_path, filename)
        self.assertPathCalledWith(self.mock_is_open, (expected_src_path,))
        self.assertPathCalledWith(self.mock_copyfile, (expected_src_path, expected_dest_path))
        self.mock_dbops_obj.insert_actionLog.assert_called_once()

    def test_Downloadfilenames_toprocess_ignores_temp_files(self):
        server_path, local_path, action, year = 's_path', 'l_path', 'Annual Stmt - Sch D', '2023'
        good_file, temp_file = '2023JHUSASchD.csv', '~$2023JHUSASchD.csv'
        self.mock_listdir.return_value = [temp_file, good_file]
        result = fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
        expected_dest_path = self._build_path(local_path, good_file)
        expected_src_path = self._build_path(server_path, good_file)
        self.assertEqual([force_forward_slashes(p) for p in result], [force_forward_slashes(expected_dest_path)])
        self.assertPathCalledWith(self.mock_is_open, (expected_src_path,))
        self.mock_is_open.assert_called_once()
        self.assertPathCalledWith(self.mock_copyfile, (expected_src_path, expected_dest_path))
        self.mock_copyfile.assert_called_once()
        self.mock_dbops_obj.insert_actionLog.assert_called_once()

    def test_Downloadfilenames_toprocess_ignores_wrong_type_or_extension_filter(self):
        server_path, local_path, action, year = 's_path', 'l_path', 'Annual Stmt - Sch D', '2023'
        good_file='2023JHUSASchD.csv'
        wrong_ext_file='2023JHUSASchD.xlsx'
        temp_file='~$2023JHUSASchD_temp.csv'
        self.mock_listdir.return_value = [temp_file, wrong_ext_file, good_file]

        with self.assertRaises(FileValidationException):
             fo.Downloadfilenames_toprocess(server_path, local_path, action, year)

        wrong_ext_src_path = self._build_path(server_path, wrong_ext_file)
        good_file_src_path = self._build_path(server_path, good_file)
        good_file_dest_path = self._build_path(local_path, good_file)

        self.assertEqual(self.mock_is_open.call_count, 2, "is_open call count mismatch")
        try:
             actual_is_open_args = {force_forward_slashes(os.path.normpath(c.args[0])) for c in self.mock_is_open.call_args_list}
        except (AttributeError, IndexError, TypeError) as e:
             raise AssertionError(f"Could not extract args from mock_is_open.call_args_list: {e}\nList: {self.mock_is_open.call_args_list}")
        expected_is_open_args = {force_forward_slashes(os.path.normpath(wrong_ext_src_path)), force_forward_slashes(os.path.normpath(good_file_src_path))}
        self.assertSetEqual(actual_is_open_args, expected_is_open_args, "is_open call arguments mismatch")

        self.mock_dbops_obj.BuildErrorMessage.assert_called_once()

        self.assertPathCalledWith(self.mock_copyfile, (good_file_src_path, good_file_dest_path))

        self.mock_dbops_obj.insert_actionLog.assert_called_once()


    def test_Downloadfilenames_toprocess_empty_after_initial_filter(self):
        server_path, local_path, action, year = 's_path', 'l_path', 'Annual Stmt - Sch D', '2023'
        files_to_list = [
            '~$tempfile.csv',
            'some_other_file.txt',
            'WrongFilenameFormat.csv',
            '2023WrongCompanySchD.csv',
            '2022JHUSASchD.csv'
        ]
        self.mock_listdir.return_value = files_to_list
        with self.assertRaises(FileValidationException):
            fo.Downloadfilenames_toprocess(server_path, local_path, action, year)

        self.assertEqual(self.mock_is_open.call_count, 2, "Expected is_open call count")
        self.mock_dbops_obj.BuildErrorMessage.assert_called_once()
        self.mock_copyfile.assert_not_called()
        self.mock_dbops_obj.insert_actionLog.assert_called_once()


    def test_Downloadfilenames_toprocess_wrong_extension_qualftc(self):
        server_path, local_path, action, year = 's_path', 'l_path', 'QualPctFTC', '2023'
        filename = '2023QualFTC.csv'
        self.mock_listdir.return_value = [filename]
        with self.assertRaises(FileValidationException):
            fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
        expected_src_path = self._build_path(server_path, filename)
        self.assertPathCalledWith(self.mock_is_open, (expected_src_path,))
        self.mock_dbops_obj.BuildErrorMessage.assert_called_once_with(f'E003,{filename},None; E002,,')
        self.mock_copyfile.assert_not_called()
        self.mock_dbops_obj.insert_actionLog.assert_called_once()

    def test_Downloadfilenames_toprocess_wrong_extension_ftcgrossup(self):
        server_path, local_path, action, year = 's_path', 'l_path', 'FTCGrossup', '2023'
        filename = '2023_Q1_FTCGrossup.csv'
        self.mock_listdir.return_value = [filename]
        with self.assertRaises(FileValidationException):
            fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
        expected_src_path = self._build_path(server_path, filename)
        self.assertPathCalledWith(self.mock_is_open, (expected_src_path,))
        self.mock_dbops_obj.BuildErrorMessage.assert_called_once_with(f'E003,{filename},None; E002,,')
        self.mock_copyfile.assert_not_called()
        self.mock_dbops_obj.insert_actionLog.assert_called_once()

if __name__ == '__main__':
    unittest.main()

