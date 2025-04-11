import unittest
from unittest.mock import MagicMock, patch, call
import os
import datetime
import shutil
import sys


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

    def _build_path(self, *args):
        return os.path.join(*args)

    def assertPathCalledWith(self, mock_func, expected_args_tuple):
        try:
            mock_func.assert_called_once()
        except AssertionError as e:
            raise AssertionError(
                f"Expected '{mock_func._extract_mock_name()}' to be called once. "
                f"Actual calls: {mock_func.call_count}\nOriginal Error: {e}"
            )

        actual_args, actual_kwargs = mock_func.call_args
        if len(actual_args) != len(expected_args_tuple):
             raise AssertionError(
                 f"Expected {len(expected_args_tuple)} positional arguments for '{mock_func._extract_mock_name()}', "
                 f"but got {len(actual_args)}. Actual args: {actual_args}"
            )

        normalized_actual = tuple(os.path.normpath(arg) if isinstance(arg, str) else arg for arg in actual_args)
        normalized_expected = tuple(os.path.normpath(arg) if isinstance(arg, str) else arg for arg in expected_args_tuple)

        self.assertEqual(normalized_actual, normalized_expected,
                         f"Normalized call args differ for '{mock_func._extract_mock_name()}':\n"
                         f"  Actual: {normalized_actual}\n"
                         f"Expected: {normalized_expected}")

    def assertPathHasCalls(self, mock_func, expected_calls_list, any_order=False):
        actual_calls = mock_func.call_args_list
        mock_name = mock_func._extract_mock_name()

        normalized_expected_calls = []
        for expected_call in expected_calls_list:
            args, kwargs = expected_call
            normalized_args = tuple(os.path.normpath(arg) if isinstance(arg, str) else arg for arg in args)
            normalized_expected_calls.append(call(*normalized_args, **kwargs))

        normalized_actual_calls = []
        for actual_call in actual_calls:
            args, kwargs = actual_call
            normalized_args = tuple(os.path.normpath(arg) if isinstance(arg, str) else arg for arg in args)
            normalized_actual_calls.append(call(*normalized_args, **kwargs))

        if any_order:
            actual_set = set(map(str, normalized_actual_calls))
            expected_set = set(map(str, normalized_expected_calls))
            if actual_set != expected_set:
                 raise AssertionError(f"Normalized calls (any order) differ for '{mock_name}':\n"
                                       f"  Actual Set: {actual_set}\n"
                                       f"Expected Set: {expected_set}\n"
                                       f"--- Missing Calls ---\n{expected_set - actual_set}\n"
                                       f"--- Unexpected Calls ---\n{actual_set - expected_set}")
        else:
             if normalized_actual_calls != normalized_expected_calls:
                  self.assertEqual(normalized_actual_calls, normalized_expected_calls,
                                   f"Normalized calls (order matters) differ for '{mock_name}':\n"
                                   f"  Actual: {normalized_actual_calls}\n"
                                   f"Expected: {normalized_expected_calls}")

    def test_getinpfilenames_toprocess(self):
        folderpath = 'test_folder'; inpLoadFolder = 'input_files'
        full_dir_path = self._build_path(folderpath, inpLoadFolder)
        self.mock_listdir.return_value = ['file1.xlsx', 'file2.xlsm', 'file3.csv', 'file4.txt']
        expected_files = [ self._build_path(full_dir_path, f) for f in ['file1.xlsx', 'file2.xlsm', 'file3.csv']]
        self.assertIs(fo.dbops_obj, self.mock_dbops_obj)
        result = fo.getinpfilenames_toprocess(folderpath, inpLoadFolder)
        self.mock_listdir.assert_called_once_with(full_dir_path)
        self.assertEqual(sorted(result), sorted(expected_files))
        self.assertNotIn(self._build_path(full_dir_path, 'file4.txt'), result)


    def test_Downloadfilenames_toprocess_annual_schd_success(self):
        server_path, local_path, action, year = 's_path', 'l_path', 'Annual Stmt - Sch D', '2023'
        filename = '2023JHUSASchD.csv'
        self.mock_listdir.return_value = [filename]
        result = fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
        expected_dest_path = self._build_path(local_path, filename)
        expected_src_path = self._build_path(server_path, filename)
        self.assertEqual(result, [expected_dest_path])
        self.assertPathCalledWith(self.mock_is_open, (expected_src_path,))
        self.assertPathCalledWith(self.mock_copyfile, (expected_src_path, expected_dest_path))
        self.mock_dbops_obj.insert_actionLog.assert_not_called()


    def test_Downloadfilenames_toprocess_qualftc_success(self):
        server_path, local_path, action, year = 's_path', 'l_path', 'QualPctFTC', '2023'
        filename = '2023QualFTC.xlsx'
        self.mock_listdir.return_value = [filename]
        result = fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
        expected_dest_path = self._build_path(local_path, filename)
        expected_src_path = self._build_path(server_path, filename)
        self.assertEqual(result, [expected_dest_path])
        self.assertPathCalledWith(self.mock_is_open, (expected_src_path,))
        self.assertPathCalledWith(self.mock_copyfile, (expected_src_path, expected_dest_path))
        self.mock_dbops_obj.insert_actionLog.assert_not_called()


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
        self.assertPathHasCalls(self.mock_is_open, expected_is_open_calls, any_order=True)
        self.assertEqual(self.mock_is_open.call_count, 4)
        self.assertPathHasCalls(self.mock_copyfile, expected_copy_calls, any_order=True)
        self.assertEqual(self.mock_copyfile.call_count, 4)
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
        self.mock_dbops_obj.insert_actionLog.assert_not_called()


    def test_Downloadfilenames_toprocess_ignores_temp_files(self):
        server_path, local_path, action, year = 's_path', 'l_path', 'Annual Stmt - Sch D', '2023'
        good_file, temp_file = '2023JHUSASchD.csv', '~$2023JHUSASchD.csv'
        self.mock_listdir.return_value = [temp_file, good_file]
        result = fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
        expected_dest_path = self._build_path(local_path, good_file)
        expected_src_path = self._build_path(server_path, good_file)
        self.assertEqual(result, [expected_dest_path])
        self.assertPathCalledWith(self.mock_is_open, (expected_src_path,))
        self.mock_is_open.assert_called_once()
        self.assertPathCalledWith(self.mock_copyfile, (expected_src_path, expected_dest_path))
        self.mock_copyfile.assert_called_once()
        self.mock_dbops_obj.insert_actionLog.assert_not_called()


    def test_Downloadfilenames_toprocess_ignores_wrong_type_or_extension_filter(self):
        server_path, local_path, action, year = 's_path', 'l_path', 'Annual Stmt - Sch D', '2023'
        good_file='2023JHUSASchD.csv'
        wrong_ext_file='2023JHUSASchD.xlsx'
        wrong_name_file='WrongFileSchD.csv'
        temp_file='~$2023JHUSASchD_temp.csv'
        self.mock_listdir.return_value = [temp_file, wrong_ext_file, wrong_name_file, good_file]
        result = fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
        expected_dest_path = self._build_path(local_path, good_file)
        expected_src_path = self._build_path(server_path, good_file)
        self.assertEqual(result, [expected_dest_path])
        self.assertPathCalledWith(self.mock_is_open, (expected_src_path,))
        self.mock_is_open.assert_called_once()
        self.assertPathCalledWith(self.mock_copyfile, (expected_src_path, expected_dest_path))
        self.mock_copyfile.assert_called_once()
        self.mock_dbops_obj.BuildErrorMessage.assert_not_called()
        self.mock_dbops_obj.insert_actionLog.assert_not_called()


    def test_Downloadfilenames_toprocess_empty_after_initial_filter(self):
        server_path, local_path, action, year = 's_path', 'l_path', 'Annual Stmt - Sch D', '2023'
        self.mock_listdir.return_value = [
            '~$tempfile.csv',
            'some_other_file.txt',
            'WrongFilenameFormat.csv',
            '2023WrongCompanySchD.csv',
            '2023JHUSASchD.xlsx'
        ]
        with self.assertRaises(FileValidationException):
            fo.Downloadfilenames_toprocess(server_path, local_path, action, year)
        self.mock_dbops_obj.BuildErrorMessage.assert_called_once_with('E002,,')
        self.mock_copyfile.assert_not_called()
        self.mock_is_open.assert_not_called()
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
