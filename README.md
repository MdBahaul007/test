import React from 'react';
import { render, fireEvent, screen, waitFor } from '@testing-library/react';
import { ComponentCodeUploadFileTemplate } from '../ComponentCodeUploadFileTemplate'; // adjust path as needed
import { componentCodeViewConfig } from '@src/utils/component-code/config/componentCodeViewConfig';
import { TemplatelploadResponseType } from '@src/utils/templateUploadResponseType';
import * as hookModule from '@src/hooks';

// Mock child components
jest.mock('@abyss/web/ui/FileUpload', () => ({
  FileUpload: ({ onChange, isUploading, headerContent }: any) => (
    <div data-testid="file-upload">
      <div data-testid="header-content">{headerContent}</div>
      <input
        data-testid="file-input"
        type="file"
        onChange={() =>
          onChange([{ name: 'dummy.csv', type: 'text/csv' }])
        }
      />
      {isUploading && <span>Uploading...</span>}
    </div>
  ),
}));

jest.mock('@abyss/web/ui/Button', () => ({
  Button: ({ onClick, children }: any) => (
    <button onClick={onClick}>{children}</button>
  ),
}));

jest.mock('@abyss/web/ui/Box', () => ({
  Box: ({ children }: any) => <div>{children}</div>,
}));

jest.mock('@abyss/web/ui/Card', () => ({
  Card: ({ children }: any) => <div>{children}</div>,
}));

jest.mock('@abyss/web/ui/LoadingOverlay', () => ({
  LoadingOverlay: () => <div>LoadingOverlay</div>,
}));

jest.mock('@src/component/shared/Alerts', () => ({
  Alerts: () => <div data-testid="alerts">Alert Component</div>,
}));

jest.mock('@src/component/result/Result', () => ({
  Result: () => <div data-testid="result-table">Result Table</div>,
}));

// Mock hook
const mockExecuteUpload = jest.fn();
const mockExecuteProcess = jest.fn();

jest.spyOn(hookModule, 'useHttpRequest').mockImplementation((apiType) => ({
  executeRequest:
    apiType === componentCodeViewConfig.apiType
      ? mockExecuteUpload
      : mockExecuteProcess,
  loading: false,
  error: null,
}));

describe('ComponentCodeUploadFileTemplate', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  test('renders the FileUpload component and simulates file upload (success path)', async () => {
    const fakeJsonData = JSON.stringify({
      groupType: 'employer',
      employerGroupComponentCodes: [
        {
          componentCodeJsonParsed: {
            code1: 'A',
            code2: 'B',
          },
        },
      ],
      componentCodeListModelId: 123,
    });

    mockExecuteUpload.mockResolvedValue({
      status: 200,
      data: {
        success: true,
        data: fakeJsonData,
      },
    });

    render(<ComponentCodeUploadFileTemplate {...componentCodeViewConfig} />);

    const fileInput = screen.getByTestId('file-input');
    fireEvent.change(fileInput);

    await waitFor(() => {
      expect(mockExecuteUpload).toHaveBeenCalled();
      expect(screen.getByTestId('result-table')).toBeInTheDocument();
    });
  });

  test('renders error alert when upload fails due to API failure', async () => {
    mockExecuteUpload.mockResolvedValue({
      status: 200,
      data: {
        success: false,
        message: 'Upload Failed',
      },
    });

    render(<ComponentCodeUploadFileTemplate {...componentCodeViewConfig} />);

    const fileInput = screen.getByTestId('file-input');
    fireEvent.change(fileInput);

    await waitFor(() => {
      expect(screen.getByTestId('alerts')).toBeInTheDocument();
    });
  });

  test('handles process record button click', async () => {
    const mockHandleTabChange = jest.fn();
    mockExecuteProcess.mockResolvedValue({ status: 200 });

    render(
      <ComponentCodeUploadFileTemplate
        {...componentCodeViewConfig}
        handleTabChange={mockHandleTabChange}
      />
    );

    const processBtn = screen.getByText('Process Record');
    fireEvent.click(processBtn);

    await waitFor(() => {
      expect(mockExecuteProcess).toHaveBeenCalled();
    });
  });

  test('handles cancel button click', async () => {
    const mockHandleTabChange = jest.fn();

    render(
      <ComponentCodeUploadFileTemplate
        {...componentCodeViewConfig}
        handleTabChange={mockHandleTabChange}
      />
    );

    const cancelBtn = screen.getByText('Cancel');
    fireEvent.click(cancelBtn);

    expect(mockHandleTabChange).toHaveBeenCalledWith(0);
  });
});
